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

#### 记录每帧刚体的速度,角速度,位置等数据
```lua
function _M:recordKeyFrame(physicsNode, frame)
    --print("try record=====", physicsNode:getName())
    if self._updateSchedule == nil or not physicsNode:getPhysicsBody():isDynamic() then
        return
    end
    local speed = print("record===============", physicsNode:getName(), frame or self._recordingFrame)
    local physicsBody = physicsNode:getPhysicsBody() --获取节点绑定的刚体
    local frameBody = {}
    local x, y = physicsNode:getPosition() --物理节点该帧的位置
    local velocity = physicsBody:getVelocity() --x,y的速度
    local angularVelocity = physicsBody:getAngularVelocity() --角速度
    frameBody.x = math.ceil(x * SYNC_PRECISION)
    frameBody.y = math.ceil(y * SYNC_PRECISION)
    --frameBody.speedX = math.ceil(velocity.x * 1000)
    --frameBody.speedY = math.ceil(velocity.y * 1000)
    frameBody.velocityLen = math.ceil(cc.pGetLength(velocity) * SYNC_PRECISION)
    frameBody.speedR = math.ceil(angularVelocity* SYNC_PRECISION)
    frameBody.frame = frame or self._recordingFrame
    local pId = physicsNode:getName()
    for k, v in pairs(self._frameCollection.data) do
        if v.id == pId then
            --只有最后一个有可能重复
            if #v.frames > 0 and v.frames[#v.frames].frame == frameBody.frame then
                v.frames[#v.frames] = frameBody
                return
            end
            table.insert(v.frames, frameBody)
            return
        end
    end
    local userFrame = {id = pId, frames = {}}
    table.insert(self._frameCollection.data, userFrame)
    table.insert(userFrame.frames, frameBody)
end
```

#### 启动定时器,记录帧数据
```lua
function _M:record()
    self._frameCollection = {data = {}, springs = {}, effs = {}, ai = aiAction}
    self._recordingFrame = 0
    --这个schedule是用来防止记录时间太长死循环的
    self._updateSchedule =
        Scheduler.scheduleGlobal(
        function()
            local startTime = os.clock()
            for i = 1, MAX_FRAME, 1 do
                local curFrame = {}
                self._recordingFrame = self._recordingFrame + 1
                self._physicsWorld:step(PHYSICAL_INTERVAL)
                self._football:check() --判断球是否停止,如果球停止了则停止定时器
                if self._football:isSleep() then
                    if self._updateSchedule then
                        Scheduler.unscheduleGlobal(self._updateSchedule)
                    end
                    print('final physical frames:=================' .. self._recordingFrame)
                    self._frameCollection.frames = self._recordingFrame
                    self:checkAndSend() --发送数据到服务,两端接受数据播放数据
                    self._updateSchedule = nil
                    self._football:clearTurn()
                    break
                end
            end
            print('calc time in sec : ', os.clock() - startTime)
            --end
        end,
        0.001
    )
end
```

#### 客户端播放根据数据,表现效果
```lua
function _M:play(frameData)
    print("============== FramePlayer frameData ="..tableplus.tostring(frameData,true))
    self:removeFrameScheduler()
    self:flush()
    self._framelen = frameData.frames
    self._curFrame = 0
    for k, v in pairs(frameData.data) do
        if self._singlePlayers[v.id] == nil then
            if v.id == "ball" then
                self._ballPlayer:play(v.frames) --播放每帧数据
            else
                print("unkown frame data========", v.id)
            end
        else
            self._singlePlayers[v.id]:play(v.frames)
        end
    end

    self._playScheduler =
        Scheduler.scheduleGlobal(
        function()
            self._curFrame = self._curFrame + 1
            self._ballPlayer:step(self._curFrame)
            for k, v in pairs(self._singlePlayers) do
                v:step(self._curFrame) --根据每帧数据,更新球的位置
            end
            for k1, v1 in pairs(frameData.springs) do
                if v1.frame == self._curFrame then
                    Game:shakeSpring(v1.id)
                end
            end
            for k2, v2 in pairs(frameData.effs) do
                if v2.frame == self._curFrame then
                    local bNeedDelay = false
                    if v2.eff == "bottlerupture" then
                        --print("------------------------------------- 播放撞击特效 ----"..v2.id)
                        local obstacleObj = Game._obstacleList:getSingleObstacleByName(v2.id)
                        obstacleObj:bottleImpact()
                        Game:playCombAnima(v2.uid)
                    elseif v2.eff == "bottleself" then
                        AudioManager.playAudio("audio/zhuangqiang.mp3", false)
                    elseif v2.eff == "bomb" then
                        --print("------------------------------------- 炸弹爆炸特效 ----"..v2.id)
                        local obstacleObj = Game._obstacleList:getSingleObstacleByName(v2.id)
                        obstacleObj:bombImpact()
                        if v2.bSuperBall then
                            self._football:changeBallSpriteFrame(Game:isMyClientRound(),false)
                        end
                        --self._framelen = self._framelen + 30
                    elseif v2.eff == "bottlebomb" then
                        local obstacleObj = Game._obstacleList:getSingleObstacleByName(v2.id)
                        obstacleObj:bottleImpact()
                        --obstacleObj:bottleBombImpact()
                        Game:playCombAnima(v2.uid)
                    elseif v2.eff == "walleff" then
                        --print("------------------------------------- 撞墙特效 ----"..tableplus.formatstring(v2.pos,true))
                        Game._wallAnima:setPosition(cc.p(v2.pos.x,v2.pos.y))
                        Game._wallAnima:setAnimation(0, "collide", false)
                        AudioManager.playAudio("audio/zhuangqiang.mp3", false)
                    elseif v2.eff == "guide_bottlerupture" then
                        Game._guide:guideBottleImpact()
                    elseif v2.eff == "bottlerupture_bysuperball" then
                        local bSuperBall = self._football:isShowSuperBallImg()
                        local obstacleObj = Game._obstacleList:getSingleObstacleByName(v2.id)
                        if bSuperBall then
                            obstacleObj:bottleBombImpact()
                        else
                            obstacleObj:bottleImpact()
                        end
                        Game:playCombAnima(v2.uid)
                    end
                end
            end
            Game:checkGameIsOver()
            if self._curFrame >= self._framelen + 30 then
                self:onPlayDone(false)
            end
        end,
        PHYSICAL_INTERVAL
    )
end
```

#### play()初始化球更新位置需要的数据,step()函数根据帧数据迭代更新球位置
```lua
function _M:play(frameCollection)
    self._finished = false
    self._bCatchBall = false
    self._frameCollection = frameCollection
    self._curIndex = 0
    self:findNextFrame()
end

function _M:findNextFrame()
    self._curIndex = self._curIndex + 1
    self._curFrame = self._frameCollection[self._curIndex]
    self._targetFrame = self._frameCollection[self._curIndex + 1]
    if self._curFrame == nil or self._targetFrame == nil then
        self._finished = true
        self._frameCollection = nil
        self._targetFrame = nil
        self._curFrame = nil
        if self._isBall then
            --self._player:pauseTail()
        end
        return
    end
    local velocity = cc.p(self._targetFrame.x - self._curFrame.x,self._targetFrame.y - self._curFrame.y)
    cc.pNormalize(velocity)
    cc.pMul(velocity,self._curFrame.velocityLen)

    self._curStep = 0
    self._frameCount = (self._targetFrame.frame - self._curFrame.frame)
    local duration = self._frameCount * PHYSICAL_INTERVAL

    self._offX = (self._targetFrame.x - self._curFrame.x) - velocity.x * duration
    self._baseX = velocity.x * duration

    self._offY = (self._targetFrame.y - self._curFrame.y) - velocity.y * duration
    self._baseY = velocity.y * duration

    self._angularSpeed = self._curFrame.speedR

    self._travelDistance = math.sqrt(math.pow(self._targetFrame.x - self._curFrame.x, 2) + math.pow(self._targetFrame.y - self._curFrame.y, 2))
end

function _M:step(globalStep)
    if self._finished then
        return
    end
    if globalStep == self._curFrame.frame then
        if #self._frameCollection > self._curIndex then --后面还有
            if self._isBall then
                --Game['hideFuckImg'](Game)
                --FuckUtil.playAudio('audio/kick.mp3', AUDIO_INTERVAL)
            else
                --FuckUtil.playAudio('audio/hit.mp3', AUDIO_INTERVAL)
            end
        end
    end
    if globalStep >= self._curFrame.frame then
        if self._isBall then
            --self._player:resumeTail()
        end
        self._curStep = self._curStep + 1
        self._angularSpeed = self._angularSpeed * self._angularFrameDam
        if self._curStep >= self._frameCount then
            self._player:updatePos(self._targetFrame.x, self._targetFrame.y, self._angularSpeed)
            self:contains(self._player,true)
            --self:findNextFrame()
        else
            local stepRatio = self._curStep / self._frameCount
            local traveledX = self._baseX * stepRatio + self._offX * math.pow(stepRatio, 1.4)
            local traveledY = self._baseY * stepRatio + self._offY * math.pow(stepRatio, 1.4)
            if math.sqrt(math.pow(traveledX, 2) + math.pow(traveledY, 2)) >= self._travelDistance then
                ---这是一个有误差的模拟运算......这里在修正误差....
                self._player:updatePos(self._targetFrame.x, self._targetFrame.y, self._angularSpeed)
                self:contains(self._player,true)
                --self:findNextFrame()
            else
                self._player:updatePos(self._curFrame.x + traveledX, self._curFrame.y + traveledY, self._angularSpeed)
                self:contains(self._player,false)
            end
        end
    end
end

-- contains函数判断对方是否接住球，接住了停止step,没有接住播放下一帧数据,直到数据播完
function _M:contains(ball,bDoNextFrameFunc)
end
```

