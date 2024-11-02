---
title: "Behind the scenes: Building Tower Age with Stride"
author: PidzamStudios
popular: false 
image: /images/blog/2024-11-10-building-tower-age-with-stride/towerage-main-art.png
tags: ['Game', 'Multiplayer', 'Education', 'Prefabs']
---

The purpose of this blog post is to give an oversight of the more challenging aspects of developing the **Tower Age** game using the Stride Game Engine.

---

Table of Contents:

[[TOC]]

## About the game

[Tower Age](https://store.steampowered.com/app/3192610/Tower_Age/) is a tower defense game with resource management mechanics where the goal is to defend your base from enemies while keeping balance between resource gathering and defense capabilities. Upgrading existing and unlocking new buildings and towers is a crucial part of the game which will allow you to keep up with the ever growing strength and tactics of the enemy.

The tower defense logic itself is very simple and works pretty much like every tower defense game out there, however compared to the other tower defense games, Tower Age has quite a few unique mechanics and systems that make it stand out. The most notable ones are the tile hunter and maze builder game modes, as well as it's 16 PvP multiplayer. So instead of going into details of the tower defense logic for which there are plenty posts and tutorials online, this blog post will mainly focus on the implementation and challenges of the unique elements of Tower Age.

{% img-click 'Tower Age Logo' '/images/blog/2024-11-10-building-tower-age-with-stride/towerage-main-art-large.png' %}

## Enemy management

As can be seen [at the end of the trailer](https://www.youtube.com/watch?v=Lc2DKoXn9LI), the game can handle hundreds of animated enemies at the same time at high FPS.

{% img-click 'Tower Age Logo' '/images/blog/2024-11-10-building-tower-age-with-stride/enemies-gif.gif' %}

Furthermore, all of these enemies can change color depending on what effect they have on them (green for poison, white for stun). This isn't as impressive as "They are Billions" for example, but keep in mind that in a blog post from them they initially tried to implement their game in Unity and there they couldn't even get a 1000 enemies at 60 FPS.

What I think is impressive about this number is how easy it is to implement this using features from the Stride Game Engine, namely the GPU instancing and sprite instancing features. The setup here is usage of GPU instancing for the models and sprite instancing for the HP bars.

For more details on these features, check out these links:

- https://github.com/tebjan/Stride.CustomRootRenderFeature
- https://github.com/tebjan/StrideTransformationInstancing

## Tile hunter

Instead of pre-created maps, in tile hunter the world is generated on the run. Here, the player starts on a small island and after every couple of waves they are given a selection of tiles to put down, thus expanding the world and path which enemies need to travel. The game mode also contains some roguelike elements which can either improve or hinder the progress the player has made so far.

{% img-click 'Tower Age Logo' '/images/blog/2024-11-10-building-tower-age-with-stride/tilehunter-gif.gif' %}

### Tile generation

The tiles themselves are pre made models in blender, which are turned into [prefabs](https://doc.stride3d.net/latest/en/manual/game-studio/prefabs/index.html) through the Game Studio editor. There are a multitude of reasons as to why it's done this way, but probably the two notable ones are difficulty control and time.

### Difficulty control

Difficulty control means controlling tile shape depending on difficulty. As you can imagine, coming up with an algorithm that can generate "easier" and "harder" tiles is not a trivial task (just defining what tile is "easy" and "hard" is difficult on it's own), and allowing any tile to spawn in any difficulty is also not a good idea firstly because complex tiles can lead to situations where it's difficult for player to put new tiles (which can lead to a game over) and secondly due to balancing which is explained later.

### Time

On the other hand, time simply means the amount of time that is required to implement the desired logic. As already said, an algorithm that would dynamically create tiles taking into account all the requirements for the game mode is not easy to come up with and implement and even as such would probably be very error prone which would require additional time spent on improvements and bugfixes. 

In contrast to the actual method used, for someone who works pretty efficiently in blender, it takes about 2-3 minutes per tile model and maybe additional 2 minutes in the Game Studio editor creating the prefab which for a 100 different tiles would be about 8 hours of work, which further means that pretty much all the tile variations can be done in a single day, keeping in mind that the potential for error is very small and would eventually require fixing the prefab.

When it comes to the prefabs, they simply contain the tile model with empty entities to mark spawn and way points within the tile. These are then later used to adjust enemy pathing as well as to define the position of adjoining tiles.

### Additional balancing

Since the path constantly expands, without additional balancing the game would eventually become too easy due to the path the enemy has to traverse becoming too long. In order to adjust for this, two balancing methods are used:

- As path expands enemies gain speed. However the intersection tiles have also to be taken into account to not go too far with the speed, but on the other hand what the player can do is keep the path lengths variable and place all turrets at the first couple of tiles. 
This way the enemies will be way too far split and will never come in groups which lets the player fight them one by one, which then again causes the game to eventually become too easy. Because of the previous problem another balancing tactic was introduced.

- If player doesn't keep all paths equal length, the enemies that spawn on the longest path get an additional HP (Hit Points?) boost that depends on the difference in path length between all paths enemies can take. This way the player can still split enemy groups, however enemies that are too far away will be way tougher to deal with. 
With this addition the player is forced to either clump up all his turrets at the first couple of tiles but fight larger groups of enemies, or spread turrets across the map but fight enemies isolated (or a combination of the two).

Additionally, since resources only spawn on tiles which would mean that the player would have no resources to gather at the beginning, a points system and buttons for resource creation were added as well. The resource creation cost grows exponentially fast with each resource built as it's only meant to help the player early and mid game, while for the rest of the game the player should have been selecting tiles appropriately to have all resources available.

## Maze builder

Similar to tile hunter, the player starts on a small island. However instead of the path being generated through tiles, here the player dictates the path the enemies have to take by blocking their path using buildings. In addition, this game mode also has roguelike elements in the form of "Blocks", where the player can select one of the given "Blocks" and position it somewhere. These "Blocks" can also change enemy pathing or contain special traits to help the player advance further.

{% img-click 'Tower Age Logo' '/images/blog/2024-11-10-building-tower-age-with-stride/mazebuilder-gif.gif' %}

### Pathing algorithm

For the purpose of finding the path between the enemy and the base, a custom algorithm was used. In principle, it's a BFS (Breadth First Search) algorithm that finds path from every position on the map to the base position. That data is then saved after which every enemy simply runs the path in the opposite direction to get it's pathing. However, as mentioned previously, the game can handle quite a few enemies simultaneously which would mean that if every enemy calculated it's own path, this algorithm would be quite costly and slow. That's why two optimizations were introduced into the algorithm:

- The first used optimization is the Belman's optimality principle. Since we know that BFS will generate one of the shortest paths between the enemy and the base, we can look at this path as the optimal path. What the mentioned principle states is that if the path between two points is optimal, then every subpath of that same path is also optimal. What this means in the case of the game is that if an enemy has generated it's path, and another enemy lies in that same path, that means that this other enemy can automatically take this same path without having to calculate it's own path from scratch because it's guaranteed to be the shortest path to the base. 

However applying this isn't enough in most cases, since the enemy path isn't generated in any specific order meaning that any enemy can calculate it's path first, which in the previous case would mean that the order of enemies can be reversed and still two paths would be generated, but this is easy to fix.

- The second optimization is done regards to the order of the path calculation. What is needed is that the path is generated first for the enemies that are farthest away from the base and then for the ones closer to it. 

This is done by simply counting the amount of steps when performing the BFS algorithm and by doing an additional pass through all enemies where they are sorted depending on their position relative to the result that the BFS algorithm generated. With these and the previous optimization it is guaranteed that only the minimal number of paths is calculated, and usually this number is in the order two or three, depending on how much the enemies are spread.

## Multiplayer

The PvP multiplayer which allows up to 16 player is implemented using the [Steam matchmaking SDK](https://partner.steamgames.com/doc/features/multiplayer/matchmaking), which made it a lot easier to manage data since most of the networking was already implemented through the SDK itself (lobbies, timeouts, UDP/TCP packet management etc.). Multithreading of multiplayer features was also very easily implemented thanks to the [async scripts](https://doc.stride3d.net/latest/en/manual/scripts/types-of-script.html#asynchronous-scripts) the Stride engine provides which left only deciding what and how much workload to put into each thread. However the number of players that are allowed still caused some problems, mainly with the bandwidth requirements to keep the game synchronized. 

To put it into perspective, let's say that every player can have up to a 100 enemies simultaneously. If full information was wanted, I calculated that around 120 bytes per enemy was required. That would mean that in the worst case scenario 6000 bytes would be transmitted and received every frame per player, and if the game runs on 60 FPS, then that adds up to 11520000 bytes or 11.52 Mbits per second only for the enemies at which point you might as well accuse the game of bandwidth mining. That's why certain cuts and optimizations had to be made to reduce this number.

The most basic form of optimizing that was done was decreasing the amount of data that gets sent. However this does cause some features loss, for example players can't view other players tower stats or resources, projectiles aren't visible etc., and while it does help a lot, due to the fact that the game uses 11 communication channels, this still wasn't enough. As can be seen, synchronizing the enemies is the biggest problem, which is why the main focus was on optimizing this aspect.

### Extrapolation of enemy position

To cut the size of the enemy data transmission, besides decreasing the amount of data per enemy, the intervals at which the transmissions occurred was also changed so that it doesn't happen every frame but every set amount of time. This was actually done for most transmissions but it was especially challenging for the enemies since they move. If no additional redesign was done, the enemies would just teleport from position to position and that would ruin the UX significantly. 

Fixing this can be quite tricky and one way to do it is to use extrapolation. What this means is that the enemy position is predicted until a new packet is received after which the position is refreshed with the actual data. 

Luckily, the game already used a custom pathing and movement algorithm, so also applying this algorithm to other player's enemies works very good. This does mean that if the enemy was for example slowed or stunned, it would stutter due to it being next to  impossible to predict whether the enemy is going to get slowed or stunned until the next packet. This effect however can be minimized by also transmitting the true speed of the enemy as well, as this would allow the extrapolation to predict the positioning better. To additionally decrease stutter effect, the frequency of transmissions can also be tuned. In the case of Tower Age, this is set to 0.5s, which cuts the transmission size by a factor of 30 when compared to transmitting every frame and minimizes the stutter effect to a point where it's barely visible.

### Testing multiplayer alone

Since [Steam SDK](https://partner.steamgames.com/doc/features/multiplayer) is used for multiplayer, the problem is that multiple instances of the same game cannot be run at once due to one person being able to join only one lobby, and just creating new steam accounts and sharing beta keys isn't enough because only one Steam instance can be run at once per computer.

Even though the game isn't fully made by a single person, being able to test multiplayer alone is still very crucial because this isn't our full time job and times where multiple people are free aren't common. Developing the multiplayer alone took a whole month and if we didn't find a way to test it this way, it would have probably taken a lot longer. Though multiple ways to test multiplayer alone were found, some of them are better than others. Here's a list of testing methods that we came across:

#### Virtual machines

These include VirtualBox, VMware workstation, HyperV, QEMU etc. Some of them are free and some are paid.

This was the first idea - install a virtual machine and run steam with the game there. Initially VirtualBox was tried and used for a while, however due to no GPU passthrough (meaning you can't use the GPU directly and all work has to be done on the CPU) only the menu options were testable, and even that would run on 3-4 FPS and was extremely tedious.

For the setup a new git repository was created (Bitbucket or GitLab are recommended since they offer a lot more space) where the game files were kept. The build was also configured with an additional post build event:

`xcopy "$(TargetDir)*" "Path_To_Local_Repository" /D /Y /I /E`

which copied only files that were changed by the build, and this worked very well with git. On the VM, the repository was simply cloned and the game could be run without problems.

VMware and HyperV would probably work a lot better since they have support for GPU passthrough, but what's also problematic with this method is the amount of resources each VM takes which would allow only 2 maybe 3 instances to run at the same time which was not good enough.

#### Windows sandbox

Windows sandbox requires Microsoft Windows Pro. This was the most used method to test initial implementations. This is meant to be the same as HyperV, however it seems to have better performance and doesn't require going through a windows installation. All that needs to be done is create a simple config file where all the required resources are mapped (for Tower Age it was the game itself, Steam, and `vcredist`). Here is an example config used for testing tower age:

```xml
<Configuration>
    <Networking>Default</Networking>
    <VGpu>Enable</VGpu>
    <MappedFolders>
        <MappedFolder>
            <HostFolder>Path_To_vcredist</HostFolder>
        </MappedFolder>
        <MappedFolder>
            <HostFolder>Path_To_Steam.exe</HostFolder>
            <ReadOnly>false</ReadOnly>
        </MappedFolder>
        pedFolder>
            <HostFolder>Path_To_Game.exe</HostFolder>
            <ReadOnly>false</ReadOnly>
        </MappedFolder>
    </MappedFolders>
</Configuration>
```

Though this method does have it's down sides, with the biggest one being that only one instance can be run at the same time. Another smaller problem is the need to go through the whole process of setting up the VM over again since all data gets wiped after exiting sandbox, but luckily for Tower Age, the only requirement was installing `vcredist`. Lastly, even with GPU passthrough, the performance is not perfect and in the case of Tower Age, the game runs stable at about 30 FPS which was good enough for testing. All in all, this is the best method of testing if only 2 players are required since it's the fastest to setup. 

#### Docker

Docker is free for non enterprise companies (though I'm not 100% sure on this). We haven't tested this for Tower Age due to the amount of setting up required which is why we were sceptical that it would even work. Basically a docker container with all the game requirements and Steam would need to be setup together with GPU passthrough and remote access. Considering how docker works, multiple instances should work and performance shouldn't be a problem since it works differently from the way VM's do, but as we don't have that much experience with this and because we found a better way, we decided not to dive deeper into this method.

#### Sandboxie

Sandboxie is free for personal use and paid for commercial use. Probably the best way to do it. This one works more similarly to docker, meaning it doesn't run VM's like previous methods do which further means that it has minimal system requirements. The performance of the game in our case was the same as the one that ran outside the isolated instance and it allows multiple game and steam instances to run at once. Using Sandboxie is very simple since it has documentation explaining everything and doesn't have problems with GPU passthrough or anything of that nature. Only down side for us was that it only has a yearly plan for the commercial licence, but we only needed it for 1-2 months, but even so it's pretty affordable.

## Links

- Steam page: https://store.steampowered.com/app/3192610/Tower_Age/
- Discord: https://discord.gg/rKmpHXP3KX


## Acknowledgment to the game engine

A huge thanks from us to the Stride Game Engine and it's community since without them none of this would be possible. Though this game doesn't fully test the graphical features of the engine, we think it definitely shows that big projects can be realized using it. We have been developing Tower Age for 2.5 years and throughout our development we never ran into a single feature or requirement that we had to remove or avoid due to the limitations of the engine, and all of this is with the release version of the engine. With new features still being added and new versions being released, we will definitely consider using this engine for our future projects as well.
