+++
title = "FRA Dungeon"
date = "2026-02-14T10:48:23+01:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
description = "My write up for the 2013 spring FRA dungeon challenge."
showFullContent = false
readingTime = false
hideComments = false
+++

>FRA PCAP Challenge våren 2013
>Det är inte nödvändigt att besvara flera frågor.
>1. Vad är flaggans absoluta position? Rita gärna en karta.
>2. 
>    - Vad kan man säga om mjukvaran som används?
>    - Är det möjligt att säga något om utvecklaren?
>3. (BONUS) Hur säker kan man vara på att det endast finns en flagga?

Opening up wireshark shows alot of traffic between two hosts.
![Wireshark showing traffic between two hosts](/FRA_Dungeon/DungeonWiresharkFirstOpen.png)
It actually consists of several streams, 6 to be exact. This can be found using wiresharks `conversations` tool. 
![Conversations found in pcap](/FRA_Dungeon/wiresharkconversations.png)
Now the most intersting thing is probably what is in these TCP packets. 
![Client-Server Communication](/FRA_Dungeon/playerserverdata.png)
To me this looks like somekind of communication for a game (I knew my gamer skills would come in handy one day). You can most easily tell this is the case by looking at the {{<red_text>}}player{{</red_text>}} packets in red. These packets include what looks like method requests to the {{<blue_text>}}server{{</blue_text>}}. These request such as `get_pos()`, `get_health()` and `move("west")`. All of which is very video gamey. 
It also appears that each conversation is one game played in the dungeon. Looking at the banner the player recieves it includes an instance number: `{'top': '\x1b[0;0H\x1b[1;34mFRA\x1b[2m Single User Dungeon v 0.2\x1b[0m\ninstance: Dungeon-1'}`. It also includes ansi escape codes which can be used to render color in a terminal (another indicator that this is a game).

So the goal is to analyze the program and also find the flag. What do we we know?
1. It is most likely a game.
2. The player sends a packet and the server responds with the answer.
3. Ansi escape code is used to render it in the terminal.

I began by creating seperate files for the {{<red_text>}}player{{</red_text>}} and {{<blue_text>}}server{{</blue_text>}}.
| {{<red_text>}}player{{</red_text>}} | {{<blue_text>}}server{{</blue_text>}}|
|-------------|----------|
|get_banner()|{'top': '\x1b[0;0H\x1b[1;34mFRA\x1b[2m Single User Dungeon v 0.2\x1b[0m\ninstance: Dungeon-1'}
|get_health()|{'health': 100}
|get_pos()|(20, 10)
|get_render_function()|{'function': "def c_render_tile(ttype):\n\ttiles={'the great flag!': '\\x1b[1m\\x1b[47mF\\x1b[0m', 'corridor': ' ', 'wall': '\\x1b[42m \\x1b[0m', 'health+100': '\\x1b[33m*\\x1b[0m', 'dragon': '\\x1b[1m\\x1b[45mD\\x1b[0m', 'monster2': '\\x1b[34mM\\x1b[0m', '?': '?', 'monster1': '\\x1b[34mm\\x1b[0m'}\n\treturn tiles[ttype]"}
|move("west")|ok
|get_adj_tiles()|{'west': 'health+100', 'east': 'corridor', 'north': 'corridor', 'south': 'corridor'}

These had to be lined up correctly for it to work in my code, if they didnt everything broke. I also did not know a good way to automate this so I had manualy copy pasted in the data and structured the data to align using linux commands (thanks chatgpt for the syntax for awk and all that magical stuff).

I coded my little game render in python and hardcoded the response for `get_render_function()` since it will be the same for every game instance.
```python
tiles_render = get_dict_format("{'the great flag!': '\\x1b[1m\\x1b[47mF\\x1b[0m', 'corridor': ' ', 'wall': '\\x1b[42m \\x1b[0m', 'health+100': '\\x1b[33m*\\x1b[0m', 'dragon': '\\x1b[1m\\x1b[45mD\\x1b[0m', 'monster2': '\\x1b[34mM\\x1b[0m', '?': '?', 'monster1': '\\x1b[34mm\\x1b[0m'}")
```

`get_dict_format()`just parses the json and returns it in a dictionary format using `ast.literal_eval()`. 

I then initlize an empty white dungeon map. 
```python
dungeon = [[f"\x1b[47m " for x in range(dungeon_width)]for y in range(dungeon_height)]
```
Then I just go through each {{<red_text>}}player{{</red_text>}} move, that is for example `get_pos()` and handle it.
I parse it so I get a function name and an argument value.
```python
for move_index in range(len(playerMoves)):
    response_index = player_move_index
    read_move = playerMoves[player_move_index]
    func = read_move.split("(")[0]
    arg = read_move.split("(")[1].split(")")[0].strip().strip('"')
    read_response = serverResponse[response_index]
    if func == "get_adj_tiles":
        get_adj_tiles(read_response)
    elif func == "get_current_tile":
        get_current_tile(read_response)
    elif func == "move":
        move(arg)
    elif func == "set_health":
        set_health(arg)
    elif func == "get_health":
        get_health(read_response)
    elif func == "get_pos":
        get_pos(read_response)
    elif func == "get_render_function":
        get_render_function()
    elif func == "get_banner":
        get_banner(serverResponse[response_index])
    else:
        print("Unknown function: " + func)
   
    print_dungeon()
```
Most of these functions just parse the server response or updates the player position. Get and set health does nothing but just return since they dont contribute to creating the map. 

`print_dungeon()` work by keeping one version with a indicator where the player is and one without it. The one without the player is used to update the dungeon map and is then copied and a player is added to it, this way I do not have to remember what tile is supposed to be where the player was after he moved.I included a player because I like watching it move and explore the dungeon haha. Print dungeon also prints the banner which includes which instance is currently running. 

```python
def print_dungeon():
    global current_banner
    global dungeon
    print(current_banner)
    dungeon_with_player = copy.deepcopy(dungeon)
    global player_pos
    pos = list(player_pos)
    dungeon_with_player[pos[0]][pos[1]] = "\033[92m●\033[0m"
    for row in dungeon_with_player:
        for cell in row:
            print(cell,end='')
        print()
```

Moving is done by updating the player position. 
```python
def move(direction):
    global player_pos
    player_pos = list(player_pos)
    if direction == "north":
        player_pos[1] += 1
    elif direction == "south":
        player_pos[1] += -1
    elif direction == "west":
        player_pos[0] += -1
    elif direction == "east":
        player_pos[0] += 1
    player_pos = tuple(player_pos)
```
`get_adj_tiles()`sets the adjecent tiles.
```python
def get_adj_tiles(tiles):
    global player_pos
    tiles = get_dict_format(tiles)
    north = tiles_render[tiles["north"]]
    west = tiles_render[tiles["west"]]
    south = tiles_render[tiles["south"]]
    east = tiles_render[tiles["east"]]
    set_tile_in_map(north,(player_pos[0],player_pos[1]+1)) 
    set_tile_in_map(west,(player_pos[0]-1,player_pos[1])) 
    set_tile_in_map(south,(player_pos[0],player_pos[1]-1)) 
    set_tile_in_map(east,(player_pos[0]+1,player_pos[1]))
```
Putting it all together you can watch players run around in the dungeon:

{{< video src="/FRA_Dungeon/dungeon.mp4" width="30%" height="30%" >}}

And the flag can be found where the player dies in instance 5:
![Flag Location](/FRA_Dungeon/flag_location.png)

So to answer the questions:
1. > Where is the flag?
    - See the picture above
2. > What can be said about the software used?
    I am abit unsure how to answer this but I can say this:
    - The software is client/server based.
    - Cheating would be very easy as the client sets the values for the health using a `set_health(x)` call. Not only that the client is also the one deciding that the game is over by sending a `game_over()` packet.
    - Probably being played in a terminal.
    - Traffic is being recieved on port `443` on the server but wireshark does not flag it as `HTTPS`, which is strange. 
    > Is it possible to say something about the developer?
    - I will be honest and say that I do not know.
    - Perhaps a beginner developer given how easy it is too cheat aswell as the unusual choice of port with traffic that does not seem to use `HTTPS`. 
3. > How sure can you be that there is only 1 flag?
    - You cant be! The data I used was was complete and consisted of a request from the {{<red_text>}}player{{</red_text>}} and a response from the {{<blue_text>}}server{{</blue_text>}}. These were the 5 first streams. The sixth stream however was incomplete and only contained the {{<blue_text>}}server{{</blue_text>}} responses. These responses did however not include a reference to a flag tile at all (except for how to render it) so the player did not 

# Full python code
Yeah I know I am not the best coder in the world.
```python 
import sys
import time
import ast
import copy
def print_dungeon():
    global current_banner
    global dungeon
    print(current_banner)
    dungeon_with_player = copy.deepcopy(dungeon)
    global player_pos
    pos = list(player_pos)
    dungeon_with_player[pos[0]][pos[1]] = "\033[92m●\033[0m"
    for row in dungeon_with_player:
        for cell in row:
            print(cell,end='')
        print()

def get_dict_format(to_parse):
    return ast.literal_eval(to_parse)

def get_banner(banner):
    global current_banner
    current_banner = get_dict_format(banner)["top"]
    print(current_banner)

def get_adj_tiles(tiles):
    global player_pos
    tiles = get_dict_format(tiles)
    north = tiles_render[tiles["north"]]
    west = tiles_render[tiles["west"]]
    south = tiles_render[tiles["south"]]
    east = tiles_render[tiles["east"]]
    set_tile_in_map(north,(player_pos[0],player_pos[1]+1)) 
    set_tile_in_map(west,(player_pos[0]-1,player_pos[1])) 
    set_tile_in_map(south,(player_pos[0],player_pos[1]-1)) 
    set_tile_in_map(east,(player_pos[0]+1,player_pos[1]))

def set_tile_in_map(tile_type,pos):
    #print(f"setting pos: {pos[0]},{pos[1]}")
    global dungeon
    dungeon[pos[0]][pos[1]] = tile_type

def get_current_tile(response):
    tile = get_dict_format(response)
    global player_pos
    set_tile_in_map(tiles_render[tile["current"]],player_pos)

def get_health(response):
    return

def get_pos(response):
    global player_pos
    pos = ast.literal_eval(response)
    player_pos = pos
    #print(player_pos)

def get_render_function():

    return

def move(direction):
    global player_pos
    player_pos = list(player_pos)
    if direction == "north":
        player_pos[1] += 1
    elif direction == "south":
        player_pos[1] += -1
    elif direction == "west":
        player_pos[0] += -1
    elif direction == "east":
        player_pos[0] += 1
    player_pos = tuple(player_pos)
    

def set_health(health):
    return
def gameover():
    return

if len(sys.argv) < 3:
    print("Please provide player and server action files")
    sys.exit(1)

player = sys.argv[1]
server = sys.argv[2]
player_pos = (0,0)
player_health = 0
current_banner = ""


#Always the same no need to parse
tiles_render = get_dict_format("{'the great flag!': '\\x1b[1m\\x1b[47mF\\x1b[0m', 'corridor': ' ', 'wall': '\\x1b[42m \\x1b[0m', 'health+100': '\\x1b[33m*\\x1b[0m', 'dragon': '\\x1b[1m\\x1b[45mD\\x1b[0m', 'monster2': '\\x1b[34mM\\x1b[0m', '?': '?', 'monster1': '\\x1b[34mm\\x1b[0m'}")

with open(player,"r") as p:
    playerMoves = [move.strip() for move in p.readlines() if move.strip()]
    p.close()
with open(server, "r") as s:
    serverResponse = [response.strip() for response in s.readlines() if response.strip()]
    s.close()

dungeon_height = 50
dungeon_width = 50

#Initilize dungeon
dungeon = [[f"\x1b[47m \x1b[0m" for x in range(dungeon_width)]for y in range(dungeon_height)]
for move_index in range(len(playerMoves)):
    response_index = move_index
    read_move = playerMoves[move_index]
    func = read_move.split("(")[0]
    arg = read_move.split("(")[1].split(")")[0].strip().strip('"')
    read_response = serverResponse[response_index]
    if func == "get_adj_tiles":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_adj_tiles(read_response)
    elif func == "get_current_tile":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_current_tile(read_response)
    elif func == "move":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        move(arg)
    elif func == "set_health":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        set_health(arg)
    elif func == "get_health":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_health(read_response)
    elif func == "get_pos":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_pos(read_response)
    elif func == "get_render_function":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_render_function()
    elif func == "get_banner":
        #print("called: " + func + ", response: " + serverResponse[response_index])
        get_banner(serverResponse[response_index])
    else:
        print("Unknown function: " + func)
    time.sleep(0.02)
    print_dungeon()
```