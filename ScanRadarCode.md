```lua
pi = math.pi
pi2 = pi*2

ign = input.getNumber


-- tags:
player = 1
static = 2


targets = {}
friendlies = {}

radarPos = {}

function clamp(x,l,h)return math.min(math.max(x,l),h)end
function vec(x,y,z)return{x=x or 0,y=y or 0,z=z or 0}end
function addS(A,B)return vec(A.x+B.x,A.y+B.y,A.z+B.z)end
function add(...)local temp = vec()for i,v in ipairs({...}) do temp = addS(temp, v)end return temp end
function mult(A,n)return type(A)=="table" and vec(A.x*n,A.y*n,A.z*n)or mult(n,A)end
function div(A,n)return mult(A,1/n)end
function sub(A,B)return addS(A,mult(B,-1))end
function len(A)return math.sqrt(A.x^2+A.y^2+A.z^2)end
function dist(A,B)return len(sub(A,B))end
function norm(A)return div(A,len(A))end
function dot(A,B)return A.x*B.x+A.y*B.y+A.z*B.z end

function lastVar(x,n,c)lVt=lVt or{}lVt[c]=lVt[c]or{}table.insert(lVt[c],1,x)lVt[c][n+2]=nil return(lVt[c][n+1])or x end --global variable "lVt" !
function delta(x,c)return type(x)=="table" and sub(x, lastVar(x,1,c.."D"))or x-lastVar(x,1,c.."D")end

function antiNan(x)return x==x and x or 0 end
function EMA(x,a,name)if not EMAtbl then EMAtbl={}end local y=a*x+(1-a)*(EMAtbl[name]or x)EMAtbl[name]=y return y end
function EMAext(x,a,name)if not EMAtblExtr then EMAtblExtr={}end local EMAval=EMA(x,a,name)local EMAext=EMAval+EMA(EMAval-(EMAtblExtr[name]or EMAval),a,name.."2")*(1/a)EMAtblExtr[name]=EMAval return EMAext end
function filter(x,a,name)return type(x)=="table" and vec(EMAext(x.x,a,name.."x"),EMAext(x.y,a,name.."y"),EMAext(x.z,a,name.."z"))or EMAext(x,a,name)end



function onTick()

	radarPos.x = ign(25)
	radarPos.y = ign(26)
	radarPos.z = ign(27)

    --IFF tracking
	local friendlyID = ign(32)
	local friendlyPos = vec(ign(29),ign(30),ign(31))
	friendlies[friendlyID] = {}
	friendlies[friendlyID].pos = friendlyPos
	friendlies[friendlyID].t = 0

	for i,v in pairs(friendlies) do
		v.t = v.t + 1
		if v.t>61 then friendlies[i] = nil end
	end



	contacts = {}
	for i=0, 7 do
		x,y,z = ign(i*3+1), ign(i*3+2), ign(i*3+3)
		if dist(vec(x,y,z),vec(ign(25),ign(26),0))>100 then
			contacts[i+1] = {}
			contacts[i+1].x = x
			contacts[i+1].y = y
			contacts[i+1].z = z
			
		end
	end
	
	debug.log(#contacts)
	
	for i,contact in pairs(contacts) do
		local closestTargetId = 0
		closestDist = 1000000
		for j,v in pairs(targets) do
			local dist = dist(v.pos,contact)
			if dist < closestDist then
				closestTargetId = j
				closestDist = dist
			end
		end

		local closestFriendId = 0
		closestFriendDist = 1000000
		for j,v in pairs(friendlies) do
			local dist = dist(v.pos,contact)
			if dist < closestFriendDist then
				closestFriendId = j
				closestFriendDist = dist
			end
		end
		
		local target = targets[closestTargetId]
		local pos = vec(contact.x,contact.y,contact.z)
        if closestFriendDist > 300 then
		    if closestTargetId ~= 0 and closestDist < (contact.z > 200 and 1500 or 800) then
		    	-- update
		    	if targets[closestTargetId].t > 0 then
		    		targets[closestTargetId].pos = pos
		    		targets[closestTargetId].dist = dist(pos, radarPos)
                    targets[closestTargetId].lastT = target.t
		    		targets[closestTargetId].t = 0
		    	end
		    else
		    	-- add
		    	local id = #targets+1
		    	targets[id] = {pos=pos, dist=dist(pos, radarPos), lastT=60, t=0, velT=0, tag=static, staticTimer=0, dirPurs=0, threshCount1=0, threshCount2=0, smoothPos=pos}
		    end
        end
	end
	
	for i,v in pairs(targets) do
	
		
		if v.velT > 90 and v.t == 0 then
			v.vel = sub(v.pos, v.lastPos or v.pos)
			v.lastPos = v.pos
			
			local x = dot(norm(v.vel),norm(v.lastVel or v.vel))
			
			if x==x then
				v.dirPurs = EMA(x,0.05,"dir"..i)
			end
			v.lastVel = v.vel
			
			if (v.dirPurs > 0.4 and v.dist < 19000) or v.pos.z > 300 then
				v.tag = player
			else
				v.tag = static
			end
			
			
			v.velT = 0
		end
		
		if v.t == 0 and v.tag == player then
			v.smoothVel = filter(div(delta(v.pos, "next"..i),v.lastT),0.1,"smoothVel"..i)
		end

		v.smoothPos = filter(add(v.pos,mult(v.smoothVel or vec(),v.t)),0.01,"smoothPos"..i)
		
		v.t = v.t + 1
		v.velT = v.velT + 1
		
		if v.t > 300 then
			targets[i] = nil
			if EMAtbl then
				EMAtbl["dir"..i] = nil
			end
            if lVt then
                lVt["next"..i] = nil
            end
		end
	end
	
    local mainTgt
    if not input.getBool(1) then
	    local minDist = 100000
	    minId = 0
	    for i,v in pairs(targets) do
	    	if v.tag == player and v.dist < minDist then
	    		minDist = v.dist
	    		minId = i
	    	end
	    end
	end

    mainTgt = targets[minId]

	if mainTgt then
		mainTgt.cleanPos = filter(mainTgt.pos,0.05,"main")
	
		output.setNumber(1,mainTgt.pos.x)
		output.setNumber(2,mainTgt.pos.y)
		output.setNumber(3,mainTgt.pos.z)
	end
	
	zoom = clamp(-100^(-ign(28))+1,0,0.99)
end


function onDraw()
	
	for i,v in pairs(targets) do
		--if v.tag == player or i==(minId or -1) then
			x,y = map.mapToScreen(radarPos.x, radarPos.y, (1-zoom)*50, 96, 96, v.pos.x, v.pos.y)
			if v.tag == player then screen.setColor(255,50,0) end
			if v.tag == static then screen.setColor(255,0,0) end
			if i == minId then screen.setColor(0,255,0) end
			screen.drawRectF(x,y,1,1)
		--end
	end

	for i,v in pairs(friendlies) do
    	screen.setColor(255,255,255)
    	x,y = map.mapToScreen(radarPos.x, radarPos.y, (1-zoom)*50, 96, 96, v.pos.x, v.pos.y)
    	screen.drawCircle(x,y,5)
	end
end
```