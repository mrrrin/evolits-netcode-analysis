# Evolits Netcode Analysis

I have obtained full access to the game's entire godot project - the client source code, along with all shaders, scenes and assets. \
I am able to use the Godot editor and recompile the game from source, which allows me to modify my client's behaviour. \
Please read about [protecting your game with PCK encryption](https://docs.godotengine.org/en/3.5/development/compiling/compiling_with_script_encryption_key.html)

With full access to the source code I was able to perform a surface-level analysis of the game's netcode.

I should preface this by saying that I have no malicious intent towards this game. \
All the events that unfolded were me testing and further validating some of the loopholes that could be maliciously exploited by other individuals. \
Compared to what other people might have done with this much exposure to the source code, what I did could be considered miniscule amounts of tomfoolery.

I have no idea how you devs are able to work with and expand upon this whole mess that is your codebase. \
To me it looks like the code was half-assed just to get stuff working as fast as possible.

The entire codebase is unclear, poorly structured, inconsistent and everything is just all over the place. \
It's gotten to the point that it might not even be worth refactoring all this mess.

That being said, I know that code clarity is not the biggest of concerns or priorities when it comes to game development. \
But there's one aspect that you should absolutely reconsider - your netcode.

***

Everything I've done was possible because of two very common, fundamental mistakes.

**1. Do not trust the client.**
The server should never "trust" the client in any regard, the information sent by each player should be strictly validated by the server and **never** only on the client-side (preferably on both)

**2. Authorize all of your network requests.**
Marking your functions as "remote" on the client means that, sure, the server can send RPC calls to the client, but so can other clients.
- With the `remote` keyword, every peer has equal authority and access to call such functions. \
  Meaning that a player can essentially "pose" as a server to another player and send calls directly to him.

  I'm assuming that most, if not all, the remote functions in your codebase are intended to be called from the server. \
In such case, the client should always validate whether or not the request has actually been made from the server and not another client.

  This can be done with a simple *if* statement at the beginning of each remote function. \
`if` [`get_tree().multiplayer.get_rpc_sender_id()`](https://docs.godotengine.org/en/3.5/classes/class_multiplayerapi.html?highlight=multiplayer#class-multiplayerapi-method-get-rpc-sender-id) `!= 1: return` \
Or, more preferably, by utilising the `master` and `puppet` keywords instead of `remote`.

  For more details, please [check out the official documentation](https://docs.godotengine.org/en/3.5/tutorials/networking/high_level_multiplayer.html)

***

I didn't yet dive deep into the source code, but below is a collection of all the notable bugs that I could find to this date.

- *Scenes/Login.gd*
  - `on_Register_pressed()` - When registering a new account, all the username checks are done on the client's side and are not server-sided, as they should be.

    ![](login.png)

- *BattleServer.gd*
  - There are 3 RPC calls being made to the Battle Server within the `_OnConnectionSucceeded()`, `readdy()` and `get_package()` functions.

    In each of them, the username is being passed as an argument to the server by the client. Problem being, that the client can pass any username they choose to.

    The server should not trust the client. It's username should be stored in the server on login, potentially as a dictionary of keys equal to clients' peer ids and values to their usernames. \
    This approach would make the server the absolute authority and not require any additional client arguments being passed in said RPC calls.

    This specific case might not prove to be of any harm, but this general idea also applies to different RPC calls being made in the *Server.gd* module.
  
      Each client request made to the server should be just as well validated as it is in the case of the `UserAttack()` function.

  - `loadBattle()` - I haven't looked much into this function or RPC, but I assume that it's possible to remotely start up a battle on someone's behalf, with any specified team of evolits or even  another player.

  - `notice()` - It's possible for someone to remotely show anyone a battle notice of custom contents.

  - `time()`, `YourTurn()` - I assume that it's possible for someone to control the entire state of the battle and skip all of your opponent's turns, presumably within PVP battles as well.

- *Server.gd*
  - `RequestLogIn()` - I see no issues with the RPC call itself, but there's no time restriction between sending each login request, making it possible for potential intruders to bruteforce their way into someone's account.

    In fact, for research purposes, I've decided to do just that on my very own account. I had a collection of over 1 million commonly used passwords and was able to make over 100,000 login requests in the time span of 8 hours.

    If I were to leave my computer running all day and night, this number would triple, giving us the value of 300,000. \
    On top of that, I was able to run at least 5 instances of the game - again, each cracking 300,000 passwords a day. \
    All the game instances were performing, in collection, 1,500,000 login requests a day, making it possible to **bruteforce at over 10,000,000 passwords a week**. \
    All of this is possible because of no enforced server-side delay between subsequent login requests.

  - `bossfight()` - The RPC call exposed in this function requires you to provide the `player_id` of the player to start the boss fight for. \
From further inspection, I've concluded that the player ID is nothing but the player's network peer ID. \
But, similar to the case of the Battle Server, I am able to pass a completely unrelated player's ID, allowing **me** to start a boss fight for a specific player or players with any of the bosses in this game. \
I'm sure many of you have experienced it yourselves with *Guard Ray*. \
The list of all the peers connected to the server can be retrieved from the client by using the [`get_tree().multiplayer.get_network_connected_peers()`](https://docs.godotengine.org/en/3.5/classes/class_multiplayerapi.html?highlight=multiplayer#class-multiplayerapi-method-get-network-connected-peers) function.

  - `evolve()` - It's possible for someone to remotely evolve someone else's evolit without their permission and absolutely no confirmation. Possibly wasting all of their essence. (Don't worry, I did not touch this function at all)
  - `send_offer()`, `offer_accepted()`, `update_trade()`, `trade_request_received()`, `start_trade()`, `update_trade()` - It's possible to **force** a trade between two different players. You could see this happen with the player `evelynn`, which I made to trade with the entire server. Sorry, I couldn't resist. This could pose more serious issues if the trade was to be remotely locked with none of the players giving their permission. I couldn't prove this to be possible.
  - `nickname()` - The profanity check performed whenever the player decides to change an evolit's nickname is entirely client-sided, allowing me to simply comment it out and completely bypass it. It's a similar case to the register username being validated just on the client within the *Scenes/Login.gd* script.
  - `Close_Game()` - Well, that one is pretty self-explanatory - you can remotely close someone else's game.
  - `notice()` - It's possible for someone to display a notice of custom contents to a player or players. You've seen that one in action. 
  - `receive_player_count()` - It's possible for someone to fake the player count for a player or players. I can't see this one being exploited for anything other than trolling.
  - `getmess()` - It's possible for someone to send a fake chat message to a player or players, even during PVP match. Presumably, you can fake both the player's name and the message contents itself.
  - `shop_recieve()`, `shop_recieve2()` - You can trigger a pop-up for a player or players, showing them that they have bought a set of items or evolits. I might or might not have used these for some trolling.
  - `update_currency()` - It's possible for someone (definitely not me) to fake the values shown on a player or players' currency display (that is gems, virtue, essence and gold). This change is entirely client-sided and has no effects over your actual, server-sided currency status. Some of y'all really thought its real and lost tons of resources, well that's what ya get :D
  - `update_variable()`, `update_data_from_server()` - Those two functions allow someone to remotely update **any** variable in the `Functions` autoload on someone else's machine. I have a feeling that these two might cause some security issues, but I couldn't find nothing of matter. 

- *Scenes/Player.gd*
  - The server is not validating the player's movement - it's very easy to modify the player's speed and create a speedhack. \
    Is the `speedhackdetector()` function in *spam.gd* supposed to be (seemingly) inactive?

Needless to say, it might come out that there's many more issues at play than the ones listed above.

***

*TL;DR:* Fix your game bros. \
And your codebase, please clean up your codebase.

Let's end things off with an example of how easily someone could ruin every player's experience.

```gdscript
while true:
    var peers = get_tree().multiplayer.get_network_connected_peers()
    for peer in peers:
        if peer != 1: # ignore server
            Server.rpc_id(peer, "Close_Game")
    yield(get_tree().create_timer(1), "timeout")
```

An unauthorized player remotely closing everyone's game from the client-side.
