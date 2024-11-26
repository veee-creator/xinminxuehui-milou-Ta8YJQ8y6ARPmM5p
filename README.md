
# 1\.概述


在rpc请求里，有了请求req就必然有回复resp。本文就来解析发送req的节点收到resp该怎么处理。


# 2\.handle\_peer\_resp源码解析



```
void raft_server::handle_peer_resp(ptr& resp, const ptr& err)
{
    if (err)
    {
        l_->info(sstrfmt("peer response error: %s").fmt(err->what()));
        return;
    }

    // update peer last response time
    {
        read_lock(peers_lock_);
        auto peer = peers_.find(resp->get_src());
        if (peer != peers_.end())
        {
            peer->second->set_last_resp(system_clock::now());
        }
        else
        {
            l_->info(sstrfmt("Peer %d not found, ignore the message").fmt(resp->get_src()));
            return;
        }
    }

    l_->debug(lstrfmt("Receive a %s message from peer %d with Result=%d, Term=%llu, NextIndex=%llu")
                  .fmt(
                      __msg_type_str[resp->get_type()],
                      resp->get_src(),
                      resp->get_accepted() ? 1 : 0,
                      resp->get_term(),
                      resp->get_next_idx()));

    {
        recur_lock(lock_);
        // if term is updated, no more action is required
        if (update_term(resp->get_term()))
        {
            return;
        }

        // ignore the response that with lower term for safety
        switch (resp->get_type())
        {
            case msg_type::vote_response:
                handle_voting_resp(*resp);
                break;
            case msg_type::append_entries_response:
                handle_append_entries_resp(*resp);
                break;
            case msg_type::install_snapshot_response:
                handle_install_snapshot_resp(*resp);
                break;
            default:
                l_->err(sstrfmt("Received an unexpected message %s for response, system exits.")
                            .fmt(__msg_type_str[resp->get_type()]));
                ctx_->state_mgr_->system_exit(-1);
                ::exit(-1);
                break;
        }
    }
}

```

* 1\.与rep\_handlers类似，resp\_handlers同样有一个总的处理resp的函数，通过switch\-case来分流。
* 2\.在交给具体的resp\_handlers之前，handle\_peer\_resp还更新了peer的last\_response\_time。
* 3\.如果这个resp可以更新节点的term，说明节点已经落后了，无需进行任何操作。


# 3\.handle\_voting\_resp源码解析



```
void raft_server::handle_voting_resp(resp_msg& resp)
{
    if (resp.get_term() != state_->get_term())
    {
        l_->info(sstrfmt("Received an outdated vote response at term %llu v.s. current term %llu")
                     .fmt(resp.get_term(), state_->get_term()));
        return;
    }

    if (election_completed_)
    {
        l_->info("Election completed, will ignore the voting result from this server");
        return;
    }

    if (voted_servers_.find(resp.get_src()) != voted_servers_.end())
    {
        l_->info(sstrfmt("Duplicate vote from %d for term %lld").fmt(resp.get_src(), state_->get_term()));
        return;
    }

    {
        read_lock(peers_lock_);
        voted_servers_.insert(resp.get_src());
        if (resp.get_accepted())
        {
            votes_granted_ += 1;
        }

        if (voted_servers_.size() >= (peers_.size() + 1))
        {
            election_completed_ = true;
        }

        if (votes_granted_ > (int32)((peers_.size() + 1) / 2))
        {
            l_->info(sstrfmt("Server is elected as leader for term %llu").fmt(state_->get_term()));
            election_completed_ = true;
            become_leader();
        }
    }
}

```

* 1\.if (resp.get\_term() !\= state\_\-\>get\_term())判断term是否相同，相同继续。
* 2\.判断if (election\_completed\_)选举是否完成，因为candidate只需要一半以上就会成功，所以可能出现election结束了但还收到了resp的情况。
* 3\.判断发送resp的节点在不在candidate的voted\_servers\_里面，在的话说明收到了同一个节点的两票，出错。
* 4\.如果 (voted\_servers\_.size() \>\= (peers\_.size() \+ 1\))说明选举已经结束。
* 5\.通过resp.get\_accepted()来统计自己的得票，如果超过了一半说明成功了，调用become\_leader();


# 4\.handle\_append\_entries\_resp源码解析



```
void raft_server::handle_append_entries_resp(resp_msg& resp)
{
    read_lock(peers_lock_);
    peer_itor it = peers_.find(resp.get_src());
    if (it == peers_.end())
    {
        l_->info(sstrfmt("the response is from an unkonw peer %d").fmt(resp.get_src()));
        return;
    }

    // if there are pending logs to be synced or commit index need to be advanced, continue to send appendEntries to
    // this peer
    bool need_to_catchup = true;
    ptr p = it->second;
    if (resp.get_accepted())
    {
        {
            auto_lock(p->get_lock());
            p->set_next_log_idx(resp.get_next_idx());
            p->set_matched_idx(resp.get_next_idx() - 1);
        }

        // try to commit with this response
        std::vector matched_indexes(peers_.size() + 1);
        matched_indexes[0] = log_store_->next_slot() - 1;
        int i = 1;
        for (it = peers_.begin(); it != peers_.end(); ++it, i++)
        {
            matched_indexes[i] = it->second->get_matched_idx();
        }

        std::sort(matched_indexes.begin(), matched_indexes.end(), std::greater());
        commit(matched_indexes[(peers_.size() + 1) / 2]);
        need_to_catchup = p->clear_pending_commit() || resp.get_next_idx() < log_store_->next_slot();
    }
    else
    {
        std::lock_guard guard(p->get_lock());
        if (resp.get_next_idx() > 0 && p->get_next_log_idx() > resp.get_next_idx())
        {
            // fast move for the peer to catch up
            p->set_next_log_idx(resp.get_next_idx());
        }
        else if (p->get_next_log_idx() > 0)
        {
            p->set_next_log_idx(p->get_next_log_idx() - 1);
        }
    }

    // This may not be a leader anymore, such as the response was sent out long time ago
    // and the role was updated by UpdateTerm call
    // Try to match up the logs for this peer
    if (role_ == srv_role::leader && need_to_catchup)
    {
        request_append_entries(*p);
    }
}

```

* 1\.首先判断发送resp的节点是不是还在peers\_列表里面，不在报错。
* 2\.如果resp的accepted \= 1，说明append\-entry成功了，设置match\_idx与next\_idx。
* 3\.提取peers\_列表里面所有的match\_idx，然后sort排序。
* 4\.排序后取中位数mid\_idx，说明至少有一半以上的follower都应用到了mid\_idx，将mid应用到自己（也就是leader）的状态机。
* 5\.如果没接受（accepted \= 0），那么就调整next\_idx继续逼近。(具体可看[cornerstone中msg类型解析](https://github.com))
* 6\.如果该节点还需要catch\_up，再发送一遍。（可能是该leader在很久之前给他发的req，现在节点才回复，导致节点依然落后。）


知识点：
log\_entry是先让follower应用到状态机，只有超过半数以上的都应用了，leader才会应用到自己的状态机。具体到实现，可以将所有节点的match\_idx排序然后取中位数。


# 5\.handle\_install\_snapshot\_resp源码解析



```
void raft_server::handle_install_snapshot_resp(resp_msg& resp)
{
    read_lock(peers_lock_);
    peer_itor it = peers_.find(resp.get_src());
    if (it == peers_.end())
    {
        l_->info(sstrfmt("the response is from an unkonw peer %d").fmt(resp.get_src()));
        return;
    }

    // if there are pending logs to be synced or commit index need to be advanced, continue to send appendEntries to
    // this peer
    bool need_to_catchup = true;
    ptr p = it->second;
    if (resp.get_accepted())
    {
        std::lock_guard guard(p->get_lock());
        ptr sync_ctx = p->get_snapshot_sync_ctx();
        if (sync_ctx == nilptr)
        {
            l_->info("no snapshot sync context for this peer, drop the response");
            need_to_catchup = false;
        }
        else
        {
            if (resp.get_next_idx() >= sync_ctx->get_snapshot()->size())
            {
                l_->debug("snapshot sync is done");
                ptr nil_snp;
                p->set_next_log_idx(sync_ctx->get_snapshot()->get_last_log_idx() + 1);
                p->set_matched_idx(sync_ctx->get_snapshot()->get_last_log_idx());
                p->set_snapshot_in_sync(nil_snp);
                need_to_catchup = p->clear_pending_commit() || resp.get_next_idx() < log_store_->next_slot();
            }
            else
            {
                l_->debug(sstrfmt("continue to sync snapshot at offset %llu").fmt(resp.get_next_idx()));
                sync_ctx->set_offset(resp.get_next_idx());
            }
        }
    }
    else
    {
        l_->info("peer declines to install the snapshot, will retry");
    }

    // This may not be a leader anymore, such as the response was sent out long time ago
    // and the role was updated by UpdateTerm call
    // Try to match up the logs for this peer
    if (role_ == srv_role::leader && need_to_catchup)
    {
        request_append_entries(*p);
    }
}

```

* 核心代码是，其他与上面一致。



```
if (resp.get_next_idx() >= sync_ctx->get_snapshot()->size())
            {
                l_->debug("snapshot sync is done");
                ptr nil_snp;
                p->set_next_log_idx(sync_ctx->get_snapshot()->get_last_log_idx() + 1);
                p->set_matched_idx(sync_ctx->get_snapshot()->get_last_log_idx());
                p->set_snapshot_in_sync(nil_snp);
                need_to_catchup = p->clear_pending_commit() || resp.get_next_idx() < log_store_->next_slot();
            }
            else
            {
                l_->debug(sstrfmt("continue to sync snapshot at offset %llu").fmt(resp.get_next_idx()));
                sync_ctx->set_offset(resp.get_next_idx());
            }

```

* 因为这是snapshot，不需要发idx，所以resp.get\_next\_idx()实际上是follower已经接受snapshot的offset。如果接受的offset \>\= sync\_ctx\-\>get\_snapshot()\-\>size()，说明已经完成了，设置next\_idx与match\_idx。否则继续从已经接受的offset位置继续发送。


# 6\.额外的ext\_resp处理源码解析



```
void raft_server::handle_ext_resp(ptr& resp, const ptr& err)
{
    recur_lock(lock_);
    if (err)
    {
        handle_ext_resp_err(*err);
        return;
    }

    l_->debug(lstrfmt("Receive an extended %s message from peer %d with Result=%d, Term=%llu, NextIndex=%llu")
                  .fmt(
                      __msg_type_str[resp->get_type()],
                      resp->get_src(),
                      resp->get_accepted() ? 1 : 0,
                      resp->get_term(),
                      resp->get_next_idx()));

    switch (resp->get_type())
    {
        case msg_type::sync_log_response:
            if (srv_to_join_)
            {
                // we are reusing heartbeat interval value to indicate when to stop retry
                srv_to_join_->resume_hb_speed();
                srv_to_join_->set_next_log_idx(resp->get_next_idx());
                srv_to_join_->set_matched_idx(resp->get_next_idx() - 1);
                sync_log_to_new_srv(resp->get_next_idx());
            }
            break;
        case msg_type::join_cluster_response:
            if (srv_to_join_)
            {
                if (resp->get_accepted())
                {
                    l_->debug("new server confirms it will join, start syncing logs to it");
                    sync_log_to_new_srv(resp->get_next_idx());
                }
                else
                {
                    l_->debug("new server cannot accept the invitation, give up");
                }
            }
            else
            {
                l_->debug("no server to join, drop the message");
            }
            break;
        case msg_type::leave_cluster_response:
            if (!resp->get_accepted())
            {
                l_->debug("peer doesn't accept to stepping down, stop proceeding");
                return;
            }

            l_->debug("peer accepted to stepping down, removing this server from cluster");
            rm_srv_from_cluster(resp->get_src());
            break;
        case msg_type::install_snapshot_response:
        {
            if (!srv_to_join_)
            {
                l_->info("no server to join, the response must be very old.");
                return;
            }

            if (!resp->get_accepted())
            {
                l_->info("peer doesn't accept the snapshot installation request");
                return;
            }

            ptr sync_ctx = srv_to_join_->get_snapshot_sync_ctx();
            if (sync_ctx == nilptr)
            {
                l_->err("Bug! SnapshotSyncContext must not be null");
                ctx_->state_mgr_->system_exit(-1);
                ::exit(-1);
                return;
            }

            if (resp->get_next_idx() >= sync_ctx->get_snapshot()->size())
            {
                // snapshot is done
                ptr nil_snap;
                l_->debug("snapshot has been copied and applied to new server, continue to sync logs after snapshot");
                srv_to_join_->set_snapshot_in_sync(nil_snap);
                srv_to_join_->set_next_log_idx(sync_ctx->get_snapshot()->get_last_log_idx() + 1);
                srv_to_join_->set_matched_idx(sync_ctx->get_snapshot()->get_last_log_idx());
            }
            else
            {
                sync_ctx->set_offset(resp->get_next_idx());
                l_->debug(sstrfmt("continue to send snapshot to new server at offset %llu").fmt(resp->get_next_idx()));
            }

            sync_log_to_new_srv(srv_to_join_->get_next_log_idx());
        }
        break;
        case msg_type::prevote_response:
            handle_prevote_resp(*resp);
            break;
        default:
            l_->err(lstrfmt("received an unexpected response message type %s, for safety, stepping down")
                        .fmt(__msg_type_str[resp->get_type()]));
            ctx_->state_mgr_->system_exit(-1);
            ::exit(-1);
            break;
    }
}

```

在解析前我们先梳理一下调用顺序：
1\.leader向新节点发送invite\_srv\_to\_join\_cluster，新节点收到invite\_srv\_to\_join\_cluster请求后更新自己的role\_，leader\_等状态，并调用reconfigure重置cluster的config。更新完后发送join\_cluster\_response给leader。
2\.leader收到该response后调用switch\-case里面的msg\_type::join\_cluster\_response分支来处理。处理完join\_cluster\_response会调用ync\_log\_to\_new\_srv向新节点发送sync\_log\_req。
3\.新节点收到sync\_log\_req后发送sync\_log\_response给leader，leader收到后调用switch\-case里面的msg\_type::sync\_log\_response分支。



```
void raft_server::sync_log_to_new_srv(ulong start_idx)
{
    // only sync committed logs
    int32 gap = (int32)(quick_commit_idx_ - start_idx);
    if (gap < ctx_->params_->log_sync_stop_gap_)
    {
        l_->info(lstrfmt("LogSync is done for server %d with log gap %d, now put the server into cluster")
                     .fmt(srv_to_join_->get_id(), gap));
        ptr new_conf = cs_new(log_store_->next_slot(), config_->get_log_idx());
        new_conf->get_servers().insert(
            new_conf->get_servers().end(), config_->get_servers().begin(), config_->get_servers().end());
        new_conf->get_servers().push_back(conf_to_add_);
        bufptr new_conf_buf(new_conf->serialize());
        ptr entry(cs_new(state_->get_term(), std::move(new_conf_buf), log_val_type::conf));
        log_store_->append(entry);
        config_changing_ = true;
        request_append_entries();
        return;
    }

    ptr req;
    if (start_idx > 0 && start_idx < log_store_->start_index())
    {
        req = create_sync_snapshot_req(*srv_to_join_, start_idx, state_->get_term(), quick_commit_idx_);
    }
    else
    {
        int32 size_to_sync = std::min(gap, ctx_->params_->log_sync_batch_size_);
        bufptr log_pack = log_store_->pack(start_idx, size_to_sync);
        req = cs_new(
            state_->get_term(),
            msg_type::sync_log_request,
            id_,
            srv_to_join_->get_id(),
            0L,
            start_idx - 1,
            quick_commit_idx_);
        req->log_entries().push_back(
            cs_new(state_->get_term(), std::move(log_pack), log_val_type::log_pack));
    }

    srv_to_join_->send_req(req, ex_resp_handler_);
}

```

* 1\.msg\_type::sync\_log\_response类型：



```
                srv_to_join_->resume_hb_speed();
                srv_to_join_->set_next_log_idx(resp->get_next_idx());
                srv_to_join_->set_matched_idx(resp->get_next_idx() - 1);
                sync_log_to_new_srv(resp->get_next_idx());

```

收到了节点的resp后，那么给该节点添加hb任务，设置next\_idx与match\_idx。然后调用sync\_log\_to\_new\_srv再去同步一遍给该节点，直到两者数据一致，否则重复发送sync\_log\_request。（类似redis里面主从同步的时候，把主节点在主从同步时候的写入操作写入一个buffer，然后在最后再发给从节点再同步一遍。）
根据上面sync\_log\_to\_new\_srv源码我们可以看到，sync\_log不是单纯采用request\_append\_entry去数据同步。因为新加入的节点落后很多，所以leader采用的机制是先发送snapshot，直到新节点的last\_log\_idx大于leader的start\_idx，接着分情况讨论，如果两者idx的差（gap） \< ctx\_\-\>params\_\-\>log\_sync\_stop\_gap\_，说明gap不足以打包成log\_pack，则调用request\_append\_entry，否则打包成log\_pack发送。


* 2\.case msg\_type::join\_cluster\_response类型:



```
case msg_type::join_cluster_response:
            if (srv_to_join_)
            {
                if (resp->get_accepted())
                {
                    l_->debug("new server confirms it will join, start syncing logs to it");
                    sync_log_to_new_srv(resp->get_next_idx());
                }
                else
                {
                    l_->debug("new server cannot accept the invitation, give up");
                }
            }
            else
            {
                l_->debug("no server to join, drop the message");
            }
            break;

```

解析完上一个resp，这里就好理解了。如果srv\_to\_join存在则调用sync\_log\_to\_new\_srv来数据同步。


* 3\.case msg\_type::leave\_cluster\_response类型：



```
case msg_type::leave_cluster_response:
            if (!resp->get_accepted())
            {
                l_->debug("peer doesn't accept to stepping down, stop proceeding");
                return;
            }

            l_->debug("peer accepted to stepping down, removing this server from cluster");
            rm_srv_from_cluster(resp->get_src());
            break;

```

重点是rm\_srv\_from\_cluster。



```
void raft_server::rm_srv_from_cluster(int32 srv_id)
{
    ptr new_conf = cs_new(log_store_->next_slot(), config_->get_log_idx());
    for (cluster_config::const_srv_itor it = config_->get_servers().begin(); it != config_->get_servers().end(); ++it)
    {
        if ((*it)->get_id() != srv_id)
        {
            new_conf->get_servers().push_back(*it);
        }
    }

    l_->info(lstrfmt("removed a server from configuration and save the configuration to log store at %llu")
                 .fmt(new_conf->get_log_idx()));
    config_changing_ = true;
    bufptr new_conf_buf(new_conf->serialize());
    ptr entry(cs_new(state_->get_term(), std::move(new_conf_buf), log_val_type::conf));
    log_store_->append(entry);
    request_append_entries();
}

```

先把要移除的srv从leader的config移除，然后把cluster的更改写入leader的log\_store，调用request\_append\_entries广播给各个follower，达到所有节点更改的效果。


* 4\.install\_snapshot\_response类型：



```
 case msg_type::install_snapshot_response:
        {
            if (!srv_to_join_)
            {
                l_->info("no server to join, the response must be very old.");
                return;
            }

            if (!resp->get_accepted())
            {
                l_->info("peer doesn't accept the snapshot installation request");
                return;
            }

            ptr sync_ctx = srv_to_join_->get_snapshot_sync_ctx();
            if (sync_ctx == nilptr)
            {
                l_->err("Bug! SnapshotSyncContext must not be null");
                ctx_->state_mgr_->system_exit(-1);
                ::exit(-1);
                return;
            }

            if (resp->get_next_idx() >= sync_ctx->get_snapshot()->size())
            {
                // snapshot is done
                ptr nil_snap;
                l_->debug("snapshot has been copied and applied to new server, continue to sync logs after snapshot");
                srv_to_join_->set_snapshot_in_sync(nil_snap);
                srv_to_join_->set_next_log_idx(sync_ctx->get_snapshot()->get_last_log_idx() + 1);
                srv_to_join_->set_matched_idx(sync_ctx->get_snapshot()->get_last_log_idx());
            }
            else
            {
                sync_ctx->set_offset(resp->get_next_idx());
                l_->debug(sstrfmt("continue to send snapshot to new server at offset %llu").fmt(resp->get_next_idx()));
            }

            sync_log_to_new_srv(srv_to_join_->get_next_log_idx());
        }
        break;

```

(1\)因为snapshot是分段传送的，如果没有srv\_to\_join\_，则根本无法跟踪offset，因此报错。
(2\)if (sync\_ctx \=\= nilptr)与(1\)同理，必须要有sync\_ctx，否则无法跟踪offset。
(3\)在前面handle\_install\_snapshot\_resp里面我们说过resp\-\>get\_next\_idx()记录的其实是snapshot的offset，根据offset我们分两种情况，如果offset \>\= sync\_ctx\-\>get\_snapshot()\-\>size()说明snapshot已经完成了，更新next\_idx与match\_idx。否则从resp里面的offset继续同步。
(4\)安装完snapshot之后还要调用更小粒度的sync\_log\_to\_new\_srv(srv\_to\_join\_\-\>get\_next\_log\_idx())来进一步同步数据。（类似redis里面持久化先应用RDB快速同步再用AOF更细粒度同步）


* 5\.case msg\_type::prevote\_response类型：



```
case msg_type::prevote_response:
            handle_prevote_resp(*resp);
            break;

```

重点是handle\_prevote\_resp：



```
void raft_server::handle_prevote_resp(resp_msg& resp)
{
    if (!prevote_state_)
    {
        l_->info(sstrfmt("Prevote has completed, term received: %llu, current term %llu")
                     .fmt(resp.get_term(), state_->get_term()));
        return;
    }

    {
        read_lock(peers_lock_);
        bool vote_added = prevote_state_->add_voted_server(resp.get_src());
        if (!vote_added)
        {
            l_->info("Prevote has from %d has been processed.");
            return;
        }

        if (resp.get_accepted())
        {
            prevote_state_->inc_accepted_votes();
        }

        if (prevote_state_->get_accepted_votes() > (int32)((peers_.size() + 1) / 2))
        {
            l_->info(sstrfmt("Prevote passed for term %llu").fmt(state_->get_term()));
            become_candidate();
        }
        else if (prevote_state_->num_of_votes() >= (peers_.size() + 1))
        {
            l_->info(sstrfmt("Prevote failed for term %llu").fmt(state_->get_term()));
            prevote_state_->reset();  // still in prevote state, just reset the prevote state
            restart_election_timer(); // restart election timer for a new round of prevote
        }
    }
}

```

如果得到票数超过一半，成为candidate，否则再开始新一轮prevote。


# 7\.总结


* 1\.log\_entry是先让follower应用到状态机，只有超过半数以上的都应用了，leader才会应用到自己的状态机。具体到实现，可以将所有节点的match\_idx排序然后取中位数。
* 2\.对于snapshot的req与resp，可以利用idx这一项来记录offset。
* 3\.对于新节点数据同步，采用snapshot，log\_pack等方式加快数据同步。
* 4\.数据同步需要多次同步，直到粒度满足要求。
* 5\.因为snapshot是分段传送的，如果无法跟踪offset，说明resp错误。


  * [1\.概述](#tid-m6ASZs)
* [2\.handle\_peer\_resp源码解析](#tid-8HiP7k)
* [3\.handle\_voting\_resp源码解析](#tid-frEJ5r)
* [4\.handle\_append\_entries\_resp源码解析](#tid-PehHwr)
* [5\.handle\_install\_snapshot\_resp源码解析](#tid-A7aFap)
* [6\.额外的ext\_resp处理源码解析](#tid-YAjWMe):[蓝猫机场](https://fenfang.org)
* [7\.总结](#tid-d2t24s)

   \_\_EOF\_\_

   ![](https://github.com/Tomgeller)TomGeller  - **本文链接：** [https://github.com/Tomgeller/p/18567510](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
