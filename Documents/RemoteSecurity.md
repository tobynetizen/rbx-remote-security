# RemoteSecurity.md
## Introduction
Remote events are an instance in Roblox that allow communication between client and server scripts respectively.\
Below is an example of this how they operate:
```lua
--> Environment: Server
local RemoteEvent: RemoteEvent = Instance.new("RemoteEvent")
RemoteEvent.Parent = game:GetService("RepicatedStorage")
RemoteEvent.Name = "RemoteEvent"

local function OnServerEvent(Player: Player, ...: any | nil): any | nil
    print(Player.Name .. " has fired this remote event!")
end

RemoteEvent.OnServerEvent:Connect(OnServerEvent)
```
```lua
--> Environment: Client
local RemoteEvent: RemoteEvent = game:GetService("ReplicatedStorage"):FindFirstChild("RemoteEvent")
RemoteEvent:FireServer()
```

Exploiters can abuse this system in two ways: (1) script decompilation, (2) metamethod hijacking\
Below is a quick crash course on how exploiters create and use these methods.
## Script Decompilation
At **publish time**, the Roblox place file (e.g., "Place.rbxl") stores the script source as string properties.\
There is also bytecode data type in the binary format, which stores the precompiled Luau bytecode (which is accessible to the game client).\
At **runtime**, the game server compiles each script's source into bytecode using the Luau compiler (which is only present on the server), and is replicated to the client as part of the data model repication stream (i.e., received over the network).\
The client's Luau virtual machine then calls an internal deserializer that: (1) parses the binary bytecode blob, (2) reconstructs prototype structures (e.g., instruction arrays, constants, upvalue metadata, etc.), (3) creates a closure wrapping the top-level prototype.\
The virtual machine's interpreter loop then executes the instructions from the prototype's instruction array by reading/writing the register stack frame.

The exploiter would use a "get_script_bytecode" function to get the precompiled bytecode inside the protected string (which as we know is accessible on the client) by: (1) locating the script object in DataModel, (2) finding the offset of the bytecode in the script class, (3) and finally, dereferencing that pointer to read the length-prefixed bytes.

After obtaining the bytecode's raw bytes, the exploiter then decompiles this bytecode by: (1) breaking the raw bytes into a table of opcode instructions, (2) extracting the constants table (which stores strings, numbers, and import names), (3) mapping out the control flow graph (i.e., "JUMP" instructions) to find where conditionals start and end, (4) and finally, pattern matching common Luau idioms (e.g., a generic "for-loop" always uses specific registers).

An example of each function in C++ is presented below:
```cpp
std::string get_script_bytecode(uintptr_t script_ptr) {
    uintptr_t bytecode_ptr = *(uintptr_t*)(script_ptr + 0x120);
    if (!bytecode_ptr) return "";
    
    int len = *(int*)(bytecode_ptr + 0x10);
    char* data = (char*)(bytecode_ptr + 0x20);
    
    return std::string(data, len);
}
```
```cpp
int lua_decompile(lua_State* L) {
    uintptr_t script = (uintptr_t)lua_touserdata(L, 1);
    std::string bc = get_script_bytecode(script);
    std::string source = ririchi::decompile(bc);
    
    lua_pushstring(L, source.c_str());
    return 1;
}
```
## Metamethod Hijacking
Using a remote spy script, exploiters have the ability to see remote event traffic that is being transmitted between the client and the server.\
This also allows them to see the remote event location and any arguments passed to the remote events.

An example of a remote spy script in Lua is presented below:
```lua
local GameMetatable = getrawmetatable(game)
local OldNamecall = GameMetatable.__namecall
setreadyonly(GameMetatable, false)

GameMetatable.__namecall = newcclosure(function(self, ...)
    local Arguments = {...}
    local Method = getnamecallmethod()

    if Method == "FireServer" then
        print("Remote event detected!")
        print("Remote Location: " .. self:GetFullName())
        print("Remote Type: " .. Method)
        print("Remote Arguments: " .. table.unpack(Arguments))
    end

    return OldNamecall(self, table.unpack(Arguments))
end)
```

## The Solution (Perchance...?)
Normally, an exploiter would be able to just use the path of the remote's location to call the remote event normally which would simplify the process for them.\
However, as a game developer we have access to the server which allows us to compile/register remote events at runtime, which can significantly slow down (but not ultimately stop) the exploiter from using their usual remote spy method.

An example of this method is presented below:
```lua
--> Environment: Server
local ReplicatedStorage: ReplicatedStorage = game:GetService("ReplicatedStorage")
local RemoteFolder: Folder = Instance.new("Folder")
local RemoteFunction: RemoteFunction = Instance.new("RemoteFunction")
local ArgumentListKey: number = 51

RemoteFolder.Parent = ReplicatedStorage
RemoteFunction.Parent = RemoteFolder

RemoteFolder.Name = "Remotes"
RemoteFunction.Name = ""

for Index, Remote in pairs(RemoteFolder:GetChildren()) do
    if Remote.ClassName == "RemoteFunction" then
        Remote.OnServerInvoke = function(Player: Player, ...: any)
            if #({...}) == ArgumentListKey then
                local RemoteEvent = Instance.new("RemoteEvent")
                return RemoteEvent
            end
        end
    end
end
```
```lua
--> Environment: Client
local RandomHashTable: table = {
    [1]  = "31CEB6C0-888A-4CDB-ABD2-1980A1C455E1",
    [2]  = "E6493637-8F68-47DA-ACA4-FB604459A07C",
    [3]  = "28533C00-DEBB-44F3-91FA-14A088AEF4F0",
    [4]  = "728FAB66-2E30-421C-B3D5-746C5F95DA1A",
    [5]  = "61654309-F7AE-4666-9C92-5FB0E963C5E9",
    [6]  = "B20BE1FB-6490-4972-856F-A9ABEE8F0FA6",
    [7]  = "CAE1F5D2-600E-462E-8979-79A08C4574DE",
    [8]  = "42337869-8A7D-438F-8516-0BC4A5BC28A1",
    [9]  = "9BA1115F-FF64-4E78-8F41-7A28DF30293A",
    [10] = "521BABBC-70A3-4B35-B03C-A0BF54C8F10B",
    [11] = "4DBCD948-2755-4264-8ECA-65A81BFD35B8",
    [12] = "3E5B37FB-2041-4434-B0F3-9BFFBEA46CBC"
}

local PayloadTable: table = {}
local PayloadKey: number = 51

for Index = 1, PayloadKey, 1 do
    table.insert(PayloadTable, RandomHashTable[math.random(1, #RandomHashTable)])
    task.wait()
end

local ReplicatedStorage: ReplicatedStorage = game:GetService("ReplicatedStorage")
local RemoteFolder: Folder = ReplicatedStorage:FindFirstChild("Remotes")
local RemoteFunction: RemoteFunction = RemoteFolder:FindFirstChildOfClass("RemoteFunction")
local RemoteEvent: RemoteEvent = RemoteFunction:InvokeServer(table.unpack(PayloadTable))
```

This is where a DSSS implementation would help us, because modern remote spy scripts have a flaw in the sense that when an uneducated exploiter attempts to locate a remote event by using a remote spy script, they wouldn't know which remote to use because their developer console output would be flooded with proxy remotes—not to mention if they try to view the remote in the remotes folder, we could just scramble the remote names and locations which would make this process nearly impossible.

An example of that proxy remote system is presented below:
```lua
local Network: table = {}
local NetworkProxySpeed: number = 0.1

function Network:FireServer(RemoteEvent: RemoteEvent, ...)
    RemoteEvent:FireServer(...)
end

function Network:SendProxies()
    while task.wait(NetworkProxySpeed) do
        Network:FireServer(RandomHashTable[math.random(1, #RandomHashTable)])
    end
end
```

However, exploiters have a trick up their sleeve called the "debug library" which is a collection of functions that allow them to do such things like viewing garbage collection, viewing the Lua registry, and viewing stack information.\
This is why we can't stop the exploiter from getting the remote event because they could technically just search table/function upvalues in the Lua registry until they find a reference to the remote event.\
Fortunately, because the initial invoke to the server to generate the remote event requires a special key, all you would have to do is hide the remote event in use by using whatever method you choose to hide it (e.g., nested function environments).\
This would ensure that the only way for the exploiter to get a useable remote event without spending hours searching for the remote in the Lua registry, they would need to decompile the local script and search through the code to find the special key so that they can generate a remote event for themself (which can be handled by simply obfuscating your local script—e.g., encoded nested hash tables, byte-to-char conversion math, etc.).