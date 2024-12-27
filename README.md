# 项目描述

该项目是记录游戏基于 URL 查询参数（如 `渠道 ID` 和 `游戏房间 ID`）的 WebSocket 请求路由与会话亲和性问题解决的过程及实现方案。通过合理配置和负载均衡策略，确保客户端在扩容和重连的情况下，能够始终连接到相同的 Pod，保持 WebSocket 会话的稳定性与一致性。

# 背景

作为游戏提供平台方，我们为多个接入方提供游戏服务。这些接入方需要接入到我们平台上的不同游戏，并在平台中创建自己的房间，供玩家进行互动。为了节省资源，并避免每个接入方都部署独立的服务器实例，多个接入方可能会共享同一实例。具体来说，多个接入方可能会共享一个游戏服务器实例，不同的游戏实例和房间通过不同的标识符（例如 AppId 和 roomId）区分开来。

为了确保玩家能够在同一房间内与其他玩家进行游戏互动，我们依赖于接入方提供的 AppId 和 roomId，这两个参数共同构成了用户的房间身份标识。通过这些标识符，平台能够确保同一房间的所有玩家始终连接到同一个游戏实例，实现多人在线游戏的实时互动。

---
## 转向 Kubernetes 的原因

在我们最初的架构中，采用了自开发的地址分发器（routing dispatcher）来将客户端请求路由到合适的服务器实例。每个客户端连接都会携带 AppId 和 roomId 查询参数，通过这些参数，地址分发器能够识别请求属于哪个接入方、哪个房间，并将请求定向到相应的游戏实例。同时，单个服务器实例可能会运行多个不同的游戏实例，因此，客户端与服务器之间的请求路由是非常依赖这些参数的。

为了保证资源的充足并应对高并发的情况，我们在原有架构中引入了弹性伸缩机制。该机制能够根据流量的变化动态调整服务器实例的数量，从而确保足够的计算资源来支撑大量的用户连接和游戏操作。

然而，这个方案存在一些严重的缺陷，导致服务在高峰时段或者遇到突发流量时容易发生大规模的服务中断：
 - 复杂的链路与难以控制的负载均衡：由于我们结合了弹性伸缩机制，负载均衡器的调整和服务的扩展都需要通过服务商提供的接口来进行。同时，地址分发器也需要注册新的实例和调整路由。这导致了服务扩展过程中链路非常长，操作复杂，难以控制。每次负载均衡调整和服务扩展都会涉及多个步骤，增加了出错的概率，也让问题诊断变得更加困难。
 - 单服务器的高风险与资源分配不灵活：由于多个游戏实例共享同一服务器，当服务器的资源（如内存、CPU）遇到突发性需求时，可能会导致整个服务器上的多个游戏实例同时宕机，造成大规模的服务中断。这种情况通常发生在内存、CPU使用量飙升的情况下，现有方案并未能有效隔离资源的使用，从而导致一个实例的资源消耗影响到其他实例。
 - 虽然弹性伸缩可以根据负载情况增加或减少实例，但由于资源分配是通过单一的服务商接口进行控制，我们无法灵活地调整不同服务器上各个游戏实例的资源分布。这种资源管理方式使得某些服务器可能会承载过多的游戏实例，而其他服务器资源闲置，造成资源的不均衡使用。

虽然我们不熟悉 Kubernetes ，但也还是坚持转型，因为项目众多，全球客户 24 小时在线，日常的维护也已经非常困难了，唯一的解决方案似乎只有 Kubernetes。

---
## 遇到的问题

在我们尝试转向之后便遇到了一系列的问题，接下来将逐步提出并解决，以作记录，难题将在最后统一讲解，因为它们是一体的：

### 一、我们如何兼容现有的游戏，确保游戏能够无需更新便无痛接入？

由于我们客户端是通过地址分发器来获取到 WebSocket 地址，所以我们选择改造了地址分发器，并且引入了实时更新的配置，当我们所配置的游戏在 Kubernetes 准备就绪时，那么将返回客户端来自 Kubernetes 的 WebSocket 地址。

### 二、在滚动更新时候，如何确保旧版本的 POD 在还存在进行中游戏时不被销毁？

我们由于游戏是有状态的，并且一局游戏可能存在持续很久的情况，所以需要在更新后旧的 `POD` 依旧能等待到所有游戏结束后再安全的销毁。

这个问题的解决方案也非常简单，我们只需要在 `Deployment` 的 `yaml` 文件中确保 `spec.template.spec.terminationGracePeriodSeconds` 设置的时间足够长。

另外开启 `Service` 的优雅停机即可。

### 三、由于 WebSocket 无法携带 Header 的特性，我们只能依赖 URL Query 来进行判断，如何确保相同接入方的相同房间的玩家进入到同一个 POD ？（难题）

有经验的同学可能会想到，NginxIngress 中有一个注解：`nginx.ingress.kubernetes.io/upstream-hash-by`，它能够基于 URL Query 实现一致性哈希。

最开始我们也是这样想的，但是这样做会存在一个严重的问题：在 Deployment 扩缩容的时候，会造成哈希重算，这样用户在断线重连后，便无法再回到之前的 POD！

### 四、由于 WebSocket 无法携带 Header 的特性，我们如何实现会话亲和？（难题）

尽管我们解决了前一个问题，但是还面临一个非常重要的问题，`Deployment` 滚动更新后，玩家重连游戏必然会被分配到新产生的 `POD` 中，原本我们可以依赖 `Service` 本身的会话亲和来实现，但是它无法根据 URL Query，所以我们必须使用 `NginxIngress` 来实现，而 `NginxIngress` 的会话亲和需要依赖 `Cookie`，完蛋~

并且即便是实现了会话亲和，也会存在一个非常严重的问题，那就是在不同设备登录，便会无法回到更新前的 `POD` 中！

## 难题解决

在我们尝试更换多种网关无果后，只能把视野重新转向 `NginxIngress` 的实现，我们发现它采用了 `Lua` 脚本来进行动态处理，这可以在 `nginx.conf` 中的 `upstream upstream_balancer` 的注释中找到。

这让我们意识到，我们可以为其增加缓存来解决我们所面临的问题，为了能够建立缓存并且进行健康检查，我们首先需要调整 `NginxIngress` 的 `ConfigMap`：
 - allow-snippet-annotations = true
 - http-snippet = lua_shared_dict hash_cache 100m;

> 其中 `lua_shared_dict hash_cache 100m` 是为了申请一个 100m 大小的共享内存作为我们的运行时缓存。

接下来我们将目标看向 `/etc/nginx/lua/balanler.lua` 的实现，并修改 `_M.balance` 函数的代码，为其添加读取缓存的逻辑：

```lua
function _M.balance()
    local cache = ngx.shared.hash_cache
    local query_arg_1 = ngx.var.arg_1
    local query_arg_2 = ngx.var.arg_1
    local uri = ngx.var.uri

    -- 检查参数是否存在
    if not query_arg_1 or not query_arg_2 then
        query_arg_1 = "0"
        query_arg_2 = "0"
    end

    local key = uri .. ":" .. query_arg_1 .. ":" .. query_arg_2
    
    -- 查询缓存
    local backend = cache:get(key)
    if backend then
        -- 续期
        ngx.log(ngx.NOTICE, "缓存命中[" .. backend .. "] " .. key)
        ngx_balancer.set_more_tries(1)
        local ok, peerErr = ngx_balancer.set_current_peer(backend)
        if not ok then
            ngx.log(ngx.ERR, "节点不可用[" .. backend .. "] " .. key .. " error: " .. peerErr)
            cache:delete(key)
            return
        end
        ngx.log(ngx.NOTICE, "缓存续期[" .. backend .. "] " .. key .. " 时效: 3 天")
        cache:set(key, backend, 259200)
        return
    end

    -- 如果缓存未命中或后端不可用，重新分配后端
    if not backend then
        local balancer = get_balancer()
        if not balancer then
            return
        end

        local peer = balancer:balance()
        if not peer then
            ngx.log(ngx.WARN, "无可用节点[" .. balancer.name .. "] " .. key)
            return
        end

        if peer:match(PROHIBITED_PEER_PATTERN) then
            ngx.log(ngx.ERR, "尝试代理到自身[" .. peer .. "] " .. key .. " 负载均衡器: " .. balancer.name)
            ngx.log(ngx.ERR, "attempted to proxy to self, balancer: ", balancer.name, ", peer: ", peer)
            return
        end

        backend = peer
        ngx_balancer.set_more_tries(1)
        local ok, err = ngx_balancer.set_current_peer(backend)
        if not ok then
            ngx.log(ngx.ERR, "节点不可用[" .. backend .. "] " .. key .. " error: " .. err)
            cache:delete(key)
        end
        ngx.log(ngx.NOTICE, "建立缓存[" .. backend .. "] " .. key .. " 时效: 3 天")
        cache:set(key, backend, 259200)
    end
end
```

代码调整了，如何替换 `Nginx` 使用的脚本呢？很简单，通过挂载点将其覆盖即可，缺陷就是每次修改均需要重启 `NginxIngress` 的 `POD`，这个目前暂时没有很好的办法。

接下来我们便可以根据 URL Query 将相同参数的玩家放置到同一 `POD` 并且在滚动更新时也不会受到哈希重算的困扰，能够重连或中途加入之前的 `POD` 中。

问题还没有结束，由于我们没有健康检查的机制，在滚动更新后，如果旧的 `POD` 被销毁，那么之前的玩家便会因为缓存原因依旧请求旧的 `POD` 导致无法访问。

但是加入健康检查并不容易，在该函数的生命周期中是不允许发起 `HTTP` 请求的（我们之前就是这么做的），所以我们这时候需要在 `location` 时候进行检查，那么，代码如下：

```lua
access_by_lua_block {
  local dns_lookup = require("util.dns").lookup
  local http = require("resty.http")
  local unpack = unpack
  local ngx = ngx
  local string = string

  local query_arg_1 = ngx.var.arg_1
  local query_arg_2 = ngx.var.arg_2
  local uri = ngx.var.uri

  local key = uri .. ":" .. query_arg_1 .. ":" .. query_arg_2
  
  local function get_resolved_url(parsed_url)
      local scheme, host, port, path = unpack(parsed_url)
      local ip = dns_lookup(host)[1]
      return string.format("%s://%s:%s%s", scheme, ip, port, path)
  end
  
  local cache = ngx.shared.hash_cache
  local backend = cache:get(key)
  if backend then
      -- 尝试访问规则地址判断后端是否可达
      local http_cli = http.new()
      http_cli:set_timeout(500)
      local parsed_url, err = http_cli:parse_uri("http://" .. backend .. uri .. "?arg_1=" .. query_arg_1 .. "&arg_2=" .. query_arg_2)
      if not parsed_url then
          ngx.log(ngx.ERROR, "解析地址失败[" .. backend .. "] " .. key .. " error: " .. err)
          return
      end
  
      local resolved_url = get_resolved_url(parsed_url)
      local http_response
      http_response, err = http_cli:request_uri(resolved_url, {
          method = "GET"
      })
  
      if not http_response then
          ngx.log(ngx.NOTICE, "缓存失效[" .. backend .. "] " .. key .. "resolved: " .. resolved_url .. " error: " .. err)
          cache:delete(key)
      else
          ngx.log(ngx.NOTICE, "缓存健康[" .. backend .. "] " .. key .. "resolved: " .. resolved_url)
      end
  end
}
```

我们这时候也需要向 `NginxIngress` 的 `ConfigMap` 中设置 `location-snippet` 的值为上述代码即可。

---
到这里，这个问题便结束了。如果你也有类似的问题或者困扰，可以参考该方案。
