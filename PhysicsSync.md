## 物理同步

最近需求做一个基于物理碰撞的回合制联网pk游戏,核心玩法是玩家A发射球撞击玩家B的钻石,玩家B判断球的移动轨迹接球避免自己方的钻石被撞。两端玩家谁先撞完对方钻石或在回合时间内那个玩家撞击的钻石数量多,则该玩家获胜。

因为是联网pk游戏,需保证球的运动轨迹两端要一致。如果两端独立执行物理逻辑,因为浮点数原因，势必造成两端球的移动轨迹出现偏差。  
基于以上原因,通过实现物理层与表现层逻辑分离,先记录球在物理层运动轨迹的帧数据,然后两端在表现层播放该帧数据的,来实现两端球运动轨迹一致的需求。

#### 初始化物理层与表现层
```lua
function _M:onCreate()
    self._viewNode = self:seekResourceChild('container') --ui文件获取表现层节点

	  --加载物理配置数据
    cc.PhysicsShapeCache:getInstance():addShapesWithFile("physicsconfig/physicsinfo.plist")
    --create physics world
    self._physicsWorld = cc.Director:getInstance():getRunningScene():getPhysicsWorld()
    -- DebugDrawMask
    self._physicsWorld:setDebugDrawMask(
           device.platform == "windows" and cc.PhysicsWorld.DEBUGDRAW_ALL or cc.PhysicsWorld.DEBUGDRAW_NONE
    )
    self._physicsWorld:setGravity(cc.p(0, 0))
    self._physicsWorld:setAutoStep(false)
    self._physicsWorld:setSubsteps(1/(30 * (1/180.0)))
    self._physicalNode = cc.Node:create()
    self._physicalNode:setName("wall")
    //设置物理层与表现层相同的位置与大小
    self._physicalNode:setPositionY(self._viewNode:getPositionY())
    self._physicalNode:setScaleX(self._viewNode:getScaleX())
    self._physicalNode:setScaleY(self._viewNode:getScaleY())
    self.resourceNode_:addChild(self._physicalNode)
    self._physicalNode:setPosition(self._viewNode:getPosition())
    self._physicalNode:setPhysicsBody(
            cc.PhysicsBody:createEdgeBox(
                    cc.size(Game.GlobalCfg.gridWidth, Game.GlobalCfg.gridHeight),
                    {density = 1, restitution = Game.PhysicsCfg.borderRestitution, friction = 1},
                    Game.GlobalCfg.edgeWidth
            )
    )
    -- 物理碰撞事件回调
    self:addPhysicsWorldListenerEvent(self._physicalNode)
    self._football = Football.new(self._physicalNode,self._viewNode) --球初始化
end
```

#### 监听物理层碰撞事件
```lua
function _M:addPhysicsWorldListenerEvent(physicalNode)
    local function onContactBegin(contact)
        return true
    end


    local function onContactSperate(contact)
        if not self._frameCollection then return end
        local a = contact:getShapeA():getBody():getNode()
        local b = contact:getShapeB():getBody():getNode()

        self:recordKeyFrame(a)
        self:recordKeyFrame(b)

        --记录碰撞,要播放的效果
        if a:getName() == "ball" then
        	--记录撞墙帧数据
            if b:getName() == "wall" then
                local x,y = self._football._physicalNode:getPosition()

                local frameBody = {id = b:getName(), frame = self._recordingFrame, eff = 'walleff',pos = {x = math.ceil(x*SYNC_PRECISION),y = math.ceil(y*SYNC_PRECISION)},uid = iPlayerUid}
                table.insert(self._frameCollection.effs, frameBody)
            end
            --记录撞钻石帧数据
            local obstacleObj = self._obstacleList:getSingleObstacleByName(b:getName())
            if obstacleObj then
                if obstacleObj._iMainType == OBSTACLE_MAINTYPE_CONF.BOTTLE then
                    if a:getTag() == obstacleObj._obstacleNode:getTag() then
                        if self._updateSchedule == nil then
                            return
                        end
                        obstacleObj:recordPhysicsImpactCount(false)
                        local sEffectName = "bottlerupture"
                        local frameBody = {id = b:getName(), frame = self._recordingFrame, eff = sEffectName,uid = iPlayerUid}
                        table.insert(self._frameCollection.effs, frameBody)
                        self._iBottleImpactCount = self._iBottleImpactCount + 1
                        if self._iBottleImpactCount >= BOTTLE_IMPACT_LIMIT then
                            self._football:setIsSleep(true, true)
                        end
                    else
                        table.insert(self._frameCollection.effs, {id = b:getName(), frame = self._recordingFrame, eff = "bottleself",uid = iPlayerUid})
                    end
                end
            end
            self._football:addCollisionCount() --球的弹射次数。
        end
    end

    local contactListener = cc.EventListenerPhysicsContact:create()
    contactListener:registerScriptHandler(onContactBegin, cc.Handler.EVENT_PHYSICS_CONTACT_BEGIN)
    contactListener:registerScriptHandler(onContactSperate, cc.Handler.EVENT_PHYSICS_CONTACT_SEPARATE)
    local eventDispatcher = self._physicalNode:getEventDispatcher()
    eventDispatcher:addEventListenerWithSceneGraphPriority(contactListener, physicalNode)
end
```

