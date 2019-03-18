# Code a la Mode Rush Post Mortem

Thanks to the creators for yet another great codingame contest. It was very original for being the first contest that requires real cooperation. It was also very heuristics-friendly, which is good for many players. Personally I don't really like writing a heuristic bot so I went with beamsearch. I still required a ton of ifs to make things work and my code was very bloated. I ended 8th, which was far better than I thought I would, after playing around with the game for 2 days. 

I wasn't sure I was going to make legend, until I finally made my search work on thursday. And when I finally saw my bot enter the top 10 after coding some oven heuristics, I was really surprised. I know I usually make top 10, but this contest is by far the hardest I've been in. 

In this post mortem I will try mainly to explain the things that are probably unique to my bot and/or interesting to read. My bot ended at 2.5k lines, so I can't go into every detail.

## Contents

+ #### Pathfinding
+ #### Beamsearch
+ #### GameState
+ #### Move pruning
+ #### Recipe matching
+ #### Cooperation with your partner
+ #### The oven
+ #### Useful tricks and fixes

## Pathfinding

In hindsight maybe not the best way, but I used two different coordinate systems. The movable part of the map is surrounded by a rectangle of squares you can't move into, so it makes sense to use a system that starts at 1,1 for movement. But the tables have different a different system. It didn't cause many issues for me, but I probably should have kept it simple.

I fill several arrays, lists and dictionaries during the first second so I could always know the distance to every table from every square.
I did this taking into account my own position, my partner position and the destination. I also made sure I could always look up which tables are adjacent to which square and which squares are adjacent to which tables. The array-lookup key for distances would be: 
myX + myY * 9 + hisX * 9 * 5 + hisY * 9 * 5 * 9 + destX * 9 * 5 * 9 * 5 + destX * 9 * 5 * 9 * 5 * 9 
This also shows you why I made a smaller coordinate system for movement. It would give me a smaller array. 

I used a simple BFS to determine the values for the distances to put into the array.

## Beamsearch

If I was going to do a search, beamsearch seemed to be perfectly suited. After some thinking it seemed to me that it would be impossible to do a true search of the opponent. Both players will have different definitions of what a good move is. Even if you guess what your partner wants to do, it may not help to move out of his way, if he doesn't know you will do that. So he has to know you know what would be a good move for him. So you're assuming he knows you know what he wants to do. Also, what if he wants to help *you* that turn? Maybe he wants to move out of your way instead? Any wrong assumption could hurt you a lot and if you want to reach any kind of depth you will have made several assumptions in a row and each successive depth level will be less likely to play out the way you expect. This is also true if you ignore the other player, but it will end up hurting less.

What I did was to take the gamestate, generate all available moves that make sense (more on pruning later) and create a new list of states for the second depth level. Then generate new moves from those, and so on. I score each state at each level and when I reach the limit (the "width" of the beam), I sort the states by score and only keep the 1000 best ones. I used a maximum depth of 30. I am not sure 1000 and 30 are good numbers, but it seemed ok. In my case, a move is an actual output possibility like "move 1 1" or "use 0 0".

## Gamestate

I tend to try and prematurely optimize, because there often isn't time to optimize later. This led me to a really silly gamestate representation:


```C#
public class GameState
{
    public GameState parent = null; // for debug purposes, to trace back a full list of actions, dont really need it
    public int myPlayer; // my position and item in a single integer
    public int[] items = new int[39]; // all items on tables, this only needs 10 bits per item, 1 bit for each type
    public int[] recipes = new int[3]; // 1 bit for each item in the recipe, also contains the score
    public bool ovenPenaltyActive; // burning food isnt penalized if the partner has oven duty
    public int myPlate; // which plate on the table am i working on?
    public int firstAction // the root action i will eventually pick if this state is the best leaf
    public int lastAction; // the action of the previous depth level, for pruning purposes
    public int platesLeft; // how many plates in the dishwasher?
    public int score; // the evaluation score, to pick the best states in the beam
    public int nextRecipe; // integer to use with the customer list, in case a recipe gets finished during search and you need more
    public int plateLastPutDown // the itemcode of the last plate you put down
    public int ovenInfo; // this holds the timer and the item in the oven

```
As you can see, almost all ints. Horrible to maintain, but it's a habit to code for performance. It hurt more than helped, this time around. All gamestates (500k) are created at gamestart and only "activated" during the seach. So I dont create new objects during the search (as always). I usually could get between 50k and 100k sims (= newly created gamestates in the search), but later on in the week, I made my move pruning a lot heavier and it went down to 30k or so.

## Move pruning

You can probably choose to do somewhere between 1 and 20 different things each turn. There's always at least 9 squares to move to if your partner is not nearby and often more. There is also the possibility of picking up and dropping items and otherwise interacting with tables. You usually have to look at least 10 turns ahead to be sure a move leads to good things. The closest table you can use is not 10 turns away, but which square do you want to land on? The one that is closest to the table you're going to move to after that, so you need that one too. There is also waiting by the oven and such. If you keep all possible actions as options, you will have explosive branching. Heavy pruning is needed. Below an incomplete list of the most important prunings you can do. Keep in mind that the other chef is stationary:

+ Don't allow moves of less than 4 squares, unless you will do "use" right after that. I solve this by always storing the previous action. If the previous action is a move less than 4 squares, moving again is disallowed. If use is not possible, the state dies.
+ Don't allow moves that go back to a square you came from last depth level.
+ Don't allow moves to a square you could already have reached the previous turn if you did a different move then 
(for example: moving 4  to the right, then 3 to the left)
+ Don't allow waiting except if you are near an oven and there is a reason to wait.
+ Don't allow picking up of items if they are not needed for recipes allowed (see recipe matching)
+ Don't allow dropping items if they are still needed for the recipe (don't drop strawberry if you can chop it and add to plate)
+ Don't allow dropping items on outer tables if middle tables are available
+ Don't allow picking up a plate if you've dropped it during the search and haven't yet added an item to it. This is to prevent repeated drops and pickups.
+ Don't allow USE on a table if you moved last turn and you could already use the table then. That would have been a useless move. 
+ Don't pick up items you don't need.
+ Only use tables that make sense for recipes allowed to fill.

The oven is excepted from some of these rules. As you can see there are loads and loads of pruning possibilities that in almost all cases do not affect the end result and they make your search manageable. Yet I still needed to score intermediate steps to filling a recipe, because you need to keep the best moves when sorting the beam (the list of 1k gamestates). 

## Recipe matching

One of the worst things that can happen is if two plates try to fill the same order. You will be left with one useless plate. Every turn I try to match all plates in the game to the available recipes. If a certain plate can only be matched to 1 recipe, that recipe is disallowed for the other plates. So my chef will never try to fill a recipe, the other chef is already trying to fill. Also if a plate is on the tables somewhere that can only be matched to 1 recipe, I will not try to fill that order either, except by picking that plate up. Every turn, I will pick one plate to focus on during my search. This choice may change as the situation changes. So if my partner chef drops his plate next to me and I am not near my own plate, I may pick it up or add items to it. 

## Cooperation with your partner

I cooperated with my partner in 4 ways (not including the oven code):

+ I put down an item if I could not add it to my own plate that turn, but my partner chef has a communicating table with me (one we both touch) and is carrying a plate that needs this item.
+ I put down a plate if my partner chef is carrying an item I need and he can't add it to a different plate that turn.
+ I put down my own plate if it has a finished order and my partner chef is at least 2 turns closer to the window and is not carrying anything.
+ If I have to use a table next turn, I will try to move to the square furthest away from my partner chef, if it does not affect my travel time to the next two tables I use. This lowers the chance I block him. 


## The oven

The difference between a good bot and a great bot is the use of the oven. I handled it as follows: 

I checked which of the players was closer to the oven (in terms of time, including the time needed to drop an item). If I am closer, I have "oven duty", this means my search gets a huge penalty if something burns. This makes sure burning doesn't happen (as often...). If the other chef is the one that put the item in the oven (meaning an odd timer) and is also equal or closer to the oven, the oven is completely inactive in my search. I can't interact with the oven until the item is completed. The item is then removed as if it never existed. This makes sure my search will ignore the oven until it is available to me and prevents me from waiting uselessly by the oven with my partner chef, only to watch him take the croissant. 

This was all I did. I think there would have been more opportunities to avoid useless visits to the oven and some more opportunities to actually use the oven. After all, if both players get the non-oven items first, then the oven goes unused and you make inefficient use of croissant/tart baking time. You will end up with 2 plates with ice/blue/strawberry that both require a croissant and/or a tart. You'll never finish those quickly then. I was planning to try giving more score to using the oven as an intermediate step, but never got around to it. 

## Useful tricks and fixes

+ I had a lot of deadlocks. Deadlocks happen when both players keep repeating the same gamestates. There are two varieties. One is where you repeat uses on the same table (picking up and dropping a plate) and the other is where your bot is continuously moving between two squares. Deadlocks happen because your opponent does something that makes you change your mind about the next use action. This makes you change direction or repeat the use on the previous table. If both players do this, you get an infinite loop (until the game ends). To fix this I recorded the previous two gamestates and on equality between the current states and two states ago, I only allowed a move that breaks the deadlock.

+ In a very late stage of the contest, I noticed my bot would go crazy if the other player was blocking a crate. Recipes can't be filled with a stationary partner chef if the chef-statue is in front of the crate. Worse, my search would end up disallowing all remaining moves, meaning my beamsearch actually ran out of states completely. After all, if you can't finish any recipe, what's the point of going on? The states that would survive this to the greatest depth levels are the states that come from the worst allowed moves. So every time this happened, my chef would do his worst move. To solve this, I checked for these type of blocks and when they occurred, I allowed a wait action on every depth level in the search. In that case, my search would still favor the highest scoring intermediate steps, even if the recipe couldnt be finished. That solved the problem. 

+ I check for useless plates (plates that have items no recipe needs). I never touch them unless the dishwasher runs out, then I can pick one up and empty it.
+ Every item is bitcoded as follows:

```C#

public const string dishString = "DISH";
public const string iceString = "ICE_CREAM";
public const string blueString = "BLUEBERRIES";
public const string strawString = "STRAWBERRIES";
public const string doughString = "DOUGH";
public const string choppedStrawString = "CHOPPED_STRAWBERRIES";
public const string choppedDoughString = "CHOPPED_DOUGH";
public const string rawTartString = "RAW_TART";
public const string tartString = "TART";
public const string croissantString = "CROISSANT";
public const int PLATE = 1;
public const int ICE = 2;
public const int BLUE = 4;
public const int CHOPPED_STRAW = 8;
public const int CROISSANT = 16;
public const int TART = 32;
public const int STRAW = 64;
public const int DOUGH = 128;
public const int CHOPPED_DOUGH = 256;
public const int RAW_TART = 512;

public static int GetItemCode(string content)
{
    int code = 0;
    string[] split = content.Split('-');
    for (int i = 0; i < split.Length; i++)
    {
        if (split[i] == dishString)
            code |= PLATE;
        else if (split[i] == iceString)
            code |= ICE;
        else if (split[i] == blueString)
            code |= BLUE;
        else if (split[i] == choppedStrawString)
            code |= CHOPPED_STRAW;
        else if (split[i] == croissantString)
            code |= CROISSANT;
        else if (split[i] == tartString)
            code |= TART;
        else if (split[i] == strawString)
            code |= STRAW;
        else if (split[i] == doughString)
            code |= DOUGH;
        else if (split[i] == choppedDoughString)
            code |= CHOPPED_DOUGH;
        else if (split[i] == rawTartString)
            code |= RAW_TART;
    }

    return code;
}

```

If you look closely, you notice the lowest 6 items are all items that may appear in a recipe, making a recipe only need 6 bits. This way it is easy to do bit operations to see which items a plate needs to fill a recipe, or if a plate is carrying items that shouldnt be there. 





