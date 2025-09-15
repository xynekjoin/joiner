-- KaitunXy – Dragon Ball Rage (con TP inicial)

local SAFE_TP=CFrame.new(0,5005,0)   -- <== cambia aquí si quieres otro punto
local TRAIN_TIME,CHARGE_TIME=60,30
local CHECK_EVERY=0.5
local OPEN_CLOSE_HOTKEY=Enum.KeyCode.RightControl
local ZENKAI_TARGETS={[19]=81e6,[20]=83e6,[21]=85e6,[22]=87e6,[23]=89e6,[24]=91e6,[25]=93e6,[26]=95e6,[27]=97e6,[28]=99e6,[29]=101e6,[30]=103e6,[31]=105e6,[32]=109e6}
local MODE_TEXTS={Combat={"combat","combate"},DefenseTrain={"defense train","defensa"},KiBlast={"ki blast","explosion ki"}}

local Players=game:GetService("Players")
local VIM=game:GetService("VirtualInputManager")
local UIS=game:GetService("UserInputService")
local CoreGui=game:GetService("CoreGui")
local SoundService=game:GetService("SoundService")
local LocalPlayer=Players.LocalPlayer

local state={running=false,target={Agility=0,Attack=0,Defense=0,Ki=0}}
local function setStatus(_)end

-- ========= INPUT / UTIL =========
local function focus3D()
  local cam=workspace.CurrentCamera
  local p=cam and cam.ViewportSize/2 or Vector2.new(200,200)
  VIM:SendMouseButtonEvent(p.X,p.Y,0,true,game,0)
  task.wait(.02)
  VIM:SendMouseButtonEvent(p.X,p.Y,0,false,game,0)
end
local function turboKey(code,total,downTime,upTime)
  downTime=downTime or 0.18; upTime=upTime or 0.05
  local t0=os.clock()
  while os.clock()-t0<(total or 1.0) do
    focus3D()
    VIM:SendKeyEvent(true,code,false,game)
    task.wait(downTime)
    VIM:SendKeyEvent(false,code,false,game)
    task.wait(upTime)
  end
end
local function clickMode(list)
  local pg=LocalPlayer:FindFirstChild("PlayerGui") if not pg then return end
  for _,d in ipairs(pg:GetDescendants())do
    if d:IsA("TextButton")then
      local t=(d.Text~=""and d.Text)or d.ContentText or"";t=string.lower(t)
      for _,s in ipairs(list)do
        if t:find(s,1,true)then pcall(function()d:Activate()end) task.wait(.15) return true end
      end
    end
  end
end

-- ========= TP SEGURO UNA SOLA VEZ =========
local didTp=false
local function ensureChar()
  if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
    LocalPlayer.CharacterAdded:Wait()
    task.wait(0.25)
  end
  return LocalPlayer.Character
end
local function teleportOnce()
  if didTp then return end
  local ch=ensureChar()
  local hrp=ch and ch:FindFirstChild("HumanoidRootPart")
  if hrp then
    hrp.CFrame=SAFE_TP
    didTp=true
    setStatus("Teleported to safe spot")
  end
end

-- ========= STATS / ZENKAI =========
local function statsFolder()
  local s=LocalPlayer:FindFirstChild("Stats") or LocalPlayer:WaitForChild("Stats",1)
  return s
end
local function readStatsCurrent()
  local s=statsFolder() if not s then return nil end
  return{Agility=s.Agility.Value,Attack=s.Attack.Value,Defense=s.Defense.Value,Ki=s.Ki.Value}
end
local function readZenkaiCurrent()
  local s=statsFolder() if not s then return nil end
  local z=s:FindFirstChild("ZenkaiBoost") return z and z.Value
end

-- ========= GUI / OVERLAY =========
local ui=Instance.new("ScreenGui") ui.Name="KaitunXy" ui.IgnoreGuiInset=true ui.ResetOnSpawn=false ui.DisplayOrder=9999 ui.Parent=CoreGui
local frame=Instance.new("Frame") frame.Size=UDim2.new(0,520,0,190) frame.Position=UDim2.new(0,100,0,100) frame.BackgroundColor3=Color3.fromRGB(25,25,25) frame.Parent=ui
local title=Instance.new("TextLabel") title.Size=UDim2.new(1,-70,0,28) title.Position=UDim2.new(0,10,0,0) title.BackgroundColor3=Color3.fromRGB(45,45,45) title.TextColor3=Color3.fromRGB(235,235,235) title.Font=Enum.Font.SourceSansBold title.TextSize=20 title.Text="KaitunXy – DB Rage" title.Parent=frame
local hideBtn=Instance.new("TextButton") hideBtn.Size=UDim2.new(0,60,0,24) hideBtn.Position=UDim2.new(1,-65,0,2) hideBtn.Text="Hide" hideBtn.Parent=frame
local statusL=Instance.new("TextLabel") statusL.BackgroundTransparency=1 statusL.Position=UDim2.new(0,10,0,40) statusL.Size=UDim2.new(1,-20,0,22) statusL.Font=Enum.Font.SourceSans statusL.TextSize=18 statusL.TextColor3=Color3.fromRGB(170,255,170) statusL.Text="Status: idle" statusL.Parent=frame
local startBtn=Instance.new("TextButton") startBtn.Text="Start" startBtn.Size=UDim2.new(0,110,0,32) startBtn.Position=UDim2.new(0,10,0,70) startBtn.Parent=frame
local stopBtn=Instance.new("TextButton") stopBtn.Text="Stop" stopBtn.Size=UDim2.new(0,110,0,32) stopBtn.Position=UDim2.new(0,140,0,70) stopBtn.Parent=frame
local overlayBtn=Instance.new("TextButton") overlayBtn.Text="Overlay" overlayBtn.Size=UDim2.new(0,120,0,32) overlayBtn.Position=UDim2.new(0,270,0,70) overlayBtn.Parent=frame
local showBtn=Instance.new("TextButton") showBtn.Size=UDim2.new(0,100,0,36) showBtn.Position=UDim2.new(0,12,0,12) showBtn.Text="Show" showBtn.Visible=false showBtn.Parent=ui
showBtn.MouseButton1Click:Connect(function()frame.Visible=true showBtn.Visible=false end)
hideBtn.MouseButton1Click:Connect(function()frame.Visible=false showBtn.Visible=true end)
UIS.InputBegan:Connect(function(i,g)if g then return end if i.KeyCode==OPEN_CLOSE_HOTKEY then frame.Visible=not frame.Visible end end)
local overlay=Instance.new("Frame") overlay.Size=UDim2.new(1,0,1,0) overlay.BackgroundColor3=Color3.new(0,0,0) overlay.Visible=false overlay.ZIndex=9997 overlay.Parent=ui
local blocker=Instance.new("TextButton") blocker.Modal=true blocker.Text="" blocker.BackgroundTransparency=1 blocker.ZIndex=9998 blocker.Size=UDim2.new(1,0,1,0) blocker.Parent=overlay
local ovTitle=Instance.new("TextButton") ovTitle.Text="KaitunXy" ovTitle.Font=Enum.Font.SourceSansBold ovTitle.TextSize=48 ovTitle.TextColor3=Color3.new(1,1,1) ovTitle.BackgroundTransparency=1 ovTitle.AnchorPoint=Vector2.new(.5,0) ovTitle.Position=UDim2.new(.5,0,0,14) ovTitle.Size=UDim2.new(0,320,0,60) ovTitle.ZIndex=9999 ovTitle.Parent=overlay
local ovStats=Instance.new("TextLabel") ovStats.Text="Stats" ovStats.Font=Enum.Font.SourceSans ovStats.TextSize=26 ovStats.TextColor3=Color3.new(1,1,1) ovStats.BackgroundTransparency=1 ovStats.AnchorPoint=Vector2.new(.5,0) ovStats.Position=UDim2.new(.5,0,0.12,0) ovStats.Size=UDim2.new(0,460,0,230) ovStats.ZIndex=9999 ovStats.Parent=overlay
local savedVolume=1
local function setOverlayEnabled(v)overlay.Visible=v blocker.Visible=v if v then savedVolume=SoundService.Volume SoundService.Volume=0 pcall(function()settings().Rendering.QualityLevel=Enum.QualityLevel.Level01 end) else SoundService.Volume=savedVolume end end
overlayBtn.MouseButton1Click:Connect(function()setOverlayEnabled(not overlay.Visible)end)
ovTitle.MouseButton1Click:Connect(function()setOverlayEnabled(false)end)

function setStatus(t)statusL.Text="Status: "..t end
local function updateOverlay()
  if not overlay.Visible then return end
  local st=readStatsCurrent()or{Agility=0,Attack=0,Defense=0,Ki=0}
  local z=readZenkaiCurrent()or 0
  local T=state.target or{}
  ovStats.Text=string.format("Agility: %d/%d\nAttack: %d/%d\nDefense: %d/%d\nKi: %d/%d\nZenkai: %d",st.Agility,T.Agility or 0,st.Attack,T.Attack or 0,st.Defense,T.Defense or 0,st.Ki,T.Ki or 0,z)
end
task.spawn(function()while true do updateOverlay() task.wait(CHECK_EVERY)end end)

-- ========= ENTRENAR =========
local function trainAttackAgility()setStatus("Train Atk/Agi")clickMode(MODE_TEXTS.Combat)turboKey(Enum.KeyCode.E,TRAIN_TIME)end
local function trainDefense()setStatus("Train Def")clickMode(MODE_TEXTS.DefenseTrain)turboKey(Enum.KeyCode.R,TRAIN_TIME)turboKey(Enum.KeyCode.C,CHARGE_TIME)end
local function trainKi()setStatus("Train Ki")clickMode(MODE_TEXTS.KiBlast)turboKey(Enum.KeyCode.Q,TRAIN_TIME)turboKey(Enum.KeyCode.C,CHARGE_TIME)end

-- ========= TARGETS =========
local function setTargetsByZenkai()
  local cur=readZenkaiCurrent() local nextZ=cur and(cur+1)
  if nextZ and ZENKAI_TARGETS[nextZ]then
    local g=ZENKAI_TARGETS[nextZ]
    state.target={Agility=g,Attack=g,Defense=g,Ki=g}
    setStatus("Targets Z"..nextZ.."->"..g)
  else state.target={Agility=81e6,Attack=81e6,Defense=81e6,Ki=81e6} end
end
local function reached(n,st)local tg=state.target[n]or 0 return(st[n]or 0)>=tg and tg>0 end
local function allReached(st)return reached("Agility",st)and reached("Attack",st)and reached("Defense",st)and reached("Ki",st)end

-- ========= LOOP PRINCIPAL =========
local function mainLoop()
  state.running=true
  teleportOnce()                         -- << TP al iniciar
  setTargetsByZenkai()
  while state.running do
    local st=readStatsCurrent()
    if st then
      if(not reached("Attack",st))or(not reached("Agility",st))then trainAttackAgility()end
      st=readStatsCurrent()or st
      if not reached("Defense",st)then trainDefense()end
      st=readStatsCurrent()or st
      if not reached("Ki",st)then trainKi()end
      st=readStatsCurrent()or st
      if allReached(st)then setStatus("All maxed – (Zenkai manual)") end
    end
    task.wait(.25)
  end
end

startBtn.MouseButton1Click:Connect(function()if state.running then return end task.spawn(mainLoop)end)
stopBtn.MouseButton1Click:Connect(function()state.running=false setStatus("Stopped")end)
task.delay(1,function()if not state.running then task.spawn(mainLoop)end end)
