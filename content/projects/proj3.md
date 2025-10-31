+++
title = "- basic persistent building in ROBLOX"
subtitle = "creating a solution to a childhood painpoint"
image = "/images/success.gif"
weight = 1
+++


**personal servers**, also known as personal building servers, were a feature on roblox that allowed players to create and share single-instance, persistent building worlds. removed on june 8th, 2016 due to moderation complexities, performance issues, and the release of team create, the once thriving community hubs--where players could do anything from host interactive game shows to found their own functioning towns--unanimously disappeared. 

![ice-castle](/images/icecastle.png)



nearly a decade later, i still remember these games. personal servers were a unique experience where emergent gameplay was king, representing a class of game whose main feature--creative engagement--was sacrificed for scalability. the hours i spent building airplanes using a physics glitch involving a toilet or putting explosives on the end of a crudely engineered catapult were experiences i still consider formative to my creative slant.

![catapult](/images/catapult.gif)

in this article i document my first steps towards a modern adaptation of the personal server -- a persistent building system.

## **building a foundation**

> while it would be nice to have a single-instance building experience, achieving true single-instance games in roblox is hacky and unreliable. instead, we'll associate build data with individual players, who will be able to load and save their same builds across instances. 

we need to store users' creations somewhere in the cloud. so we first must understand the difference between roblox's two cloud storage services: **datastores** and **memorystores.** datastores are persistent, cloud-backed storage systems designed to save data between play sessions. memorystores are a low latency, high throughput service that provides storage accessible across all servers. however, despite being fast, this storage is not designed to be large and expires, so **datastores** are the best choice. 

cool, we know exactly what we'll use to store our builds. but a datastore is essentially a dictionary, and players' builds are instances. so how do we transform one into another and vice-versa?

### **serialization**

instance serialization is the process of transforming instances into formatted text. normally, this is manually implemented by roblox devs depending on what exactly they need to be stored in the cloud. this makes sense: the less objects you need to serialize, the less cloud storage is used. but today there are better solutions to this problem (recursive ones even). one of them is the excellently made [RoSe](https://github.com/Gem-API/Rose) by Gem_API, which reduces the serialization and deserialization process to two lines:

```lua
-- To turn an Instance into a table:
local rose_table = rose.serialize(instance)

-- To turn a Rose table back into an Instance:
local result = rose.deserialize(rose_table)

```

awesome! so now that we have a serialization framework, the rest should be straightforward. 

### **packing**

for each player's build, we'll use a prefix + their unique playerid as a key to access their data. 

```lua
local players = game:GetService("Players")
local dataStoreService = game:GetService("DataStoreService")
local RS = game:GetService("ReplicatedStorage")
local dataStore = dataStoreService:GetDataStore("InstanceSave")
local rose = require(libraries.Rose)
local keyPrefix = "Player: "

-- save function begin
local function save(player: Player)
	local key = keyPrefix .. tostring(player.UserId) 
    -- our key will be "Player: userid"
	local data = {} 
    -- instantiate the data table associated with our key that we want to store
	local model = Instance.new("Model", workspace) 
    -- instantiate model to store player's saved build inside 
	model.Name = "Model" .. tostring(player.UserId) 
    -- associate the model with the player

```

we'll store a clone of all parts owned by a player in a model to give RoSe, which will recursively serialize the model's children and their properties

>to determine which parts should be serialized when a player clicks save, i brutishly modified the game's building tools to rename parts selected by the player to "Part+userid" if a name does not already exist. the loop below checks if the userid stored in the parts matches the requesting player's before saving. yuck.

```lua
    for i, obj: Instance in ipairs(parts:GetChildren()) do
            if obj.Name ~= "Part" .. tostring(player.UserId) then continue end 
            -- bad mechanism for player part ownership, modify this later..
            local clone = obj:Clone() 
            clone.Parent = model 
            -- clone each part and put it inside the model
        end

```

now we do some cleanup, making sure to store the model clone in ReplicatedStorage before using RoSe. if the serialization succeeds, we insert the data into the table we instantiated earlier.

```lua
    model.Parent = RS
     -- store build in replicated storage so it cannot be modified by player(s)
	model:PivotTo(CFrame.new(0,0,0)) 
    -- center build model's position property to origin
	local serializedData = rose.serialize(model)
    -- use RoSe to recursively serialize instance trees
    --TODO: convert to JSON and compress before storing, then convert back after
	if serializedData then
		table.insert(data, serializedData)
	else
		warn("Failed to serialize object: " .. model.Name)
	end
    -- insert serialized entry into table for storage
```

lastly, we store the data, as well as the game's jobid (which will be important later) in the cloud

```lua
    local saveData = {
            JobId = game.JobId, -- store the current instance's JobId
            Data = data         -- store serialized data
        }
        local success, err
        repeat
            success, err = pcall(function()
                dataStore:UpdateAsync(key, function()
                    return saveData
                    -- update dataStore with table of serializations and JobId
                end)
            end)
        until success
        if not success then
            warn("Failed to save data: " .. tostring(err))
        end
    end
-- save function end
```
### **unpacking**

next, we'll implement the slightly more complicated load function. while it doesn't necessarily need to take both the player and a CFrame as an argument, i built a separate script to record the player's mouse position when placing the model. the first part of the script is essentially the reverse of save()'s last bits.

```lua
local function load(player: Player, pos: CFrame)
	local key = keyPrefix .. tostring(player.UserId)
	local success, err
	local savedData

	repeat
		success, err = pcall(function()
			savedData = dataStore:GetAsync(key)
            -- retrieve data for datastore
		end)
	until success or not players:FindFirstChild(player.Name)

	if success and savedData then
		-- check if the JobId matches the current instance's JobId
		if savedData.JobId == game.JobId then
			print(player.Name .. " has already joined this instance. Halting load.")
			return -- halt the load function to prevent duplicate loads
		end

		local data = savedData.Data
```
we use the JobId stored from earlier to prevent the player from loading more than once. then, we deserialize the object and return it to the workspace, making sure to reparent all parts independently to the workspace.

```lua
        local data = savedData.Data
        local serializedModel = data and data[1]

        if serializedModel then
            local newObj = rose.deserialize(serializedModel)
            -- RoSe handles deserialization
            if newObj then
                if newObj.className == "Model" then
                    local _, size = newObj:GetBoundingBox()
                    local bottomOffset = size.Y / 2
                    --[[
            we centered the model at the origin, but in order to 
            make sure the build does not load partially underground,
            we offset Y by half the height of the model.
                    ]] 
                    newObj:PivotTo(newObj:GetPivot() + pos + Vector3.new(0, bottomOffset, 0))
                    Helper.decouple(newObj)
                    -- helper method to reparent all deserialized parts to workspace
                end
                newObj.Parent = workspace.Parts
            else
                warn("failed to deserialize object")
            end
        else
            warn("no serialized model found in saved data")
        end
    elseif not success then
		warn("failed to load data: " .. tostring(err))
	end
end
```

now we just create remotes to trigger the functions!

```lua
local loadParts = RS:FindFirstChild("Loadparts")
local saveParts = RS:FindFirstChild("Saveparts")
loadParts.OnServerEvent:Connect(load)
saveParts.OnServerEvent:Connect(save)
```


## **the finished product**

![success](/images/success(1).gif)

voila! if you have any questions, comments, or suggestions for this post, feel free to [contact me](/contact/)