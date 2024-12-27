# k8s-nginx-ingress-game-websocket-session-affinity
该仓库用于记录解决基于 URL 查询参数（如 platformid 和 roomid）的 WebSocket 请求路由与会话亲和性问题。通过合理配置和负载均衡策略，确保客户端在扩容和重连的情况下，能够始终连接到相同的 Pod，保持 WebSocket 会话的稳定性与一致性。
