+++
title = "100 hours in 10 minutes: How we test Fields of Mistria"
description = "How a test suite catches crashes, softlocks, and broken saves in a big game made by a small team."
date = 2026-10-03
+++

{{ <figure src="suite.mp4" alt="The test suite running" /> }}

[Fields of Mistria](https://store.steampowered.com/app/2142790/Fields_of_Mistria/) is, among many things, a very large video game. With a plethora of ways to play and twelve different characters to date and marry, there's an endless number of states for the game to find itself in. Players can decorate their houses however they like, dress however they please, and spend their days in whatever way they see fit. We also launched in early access two years before 1.0, with a promise to always keep save files compatible. And we support eight languages. And there's only 15 of us. Oh, and we switched from [GameMaker Studio 2](https://gamemaker.io/en) to an in-house engine a couple of months before 1.0.

Because numbers are fun, let's break some down to put the project we're covering in context.

{% <stats caption="Numbers as of the date this article was posted."> %}
{{ <stat value="140,042" label="lines of GML (gameplay scripts)" /> }}
{{ <stat value="120,453" label="lines of Rust (engine, VM)[^rust_loc]" /> }}
{{ <stat value="3,371" label="bugs fixed" /> }}
{{ <stat value="6,112" label="pull requests" /> }}
{{ <stat value="7" label="years of development" /> }}
{{ <stat value="15" label="team members, at peak[^team_members]" /> }}
{{ <stat value="52,997" label="executions of the test suite" /> }}
{{ <stat value="1" label="engine swaps" /> }}
{% </stats> %}

Games can't be tested the same way other projects can be. Unit tests don't look in the right places to find the bugs a game is likely to encounter. Don't get me wrong, I love a good unit test -- [one of my other projects has over 2,200 of them](https://mim.as). But unit tests take small, focused sections of code and test them in **isolation**. They're great at maintaining correctness within an individual system, but they are conceptually blind to the ways that systems will interact with the rest of the codebase.

The principal challenge for stability in video games is not in writing individual systems. Instead, **bugs almost always emerge where systems cross paths**, out in the "real world", where they're all playing in the same sandbox. To test this, we need an _integration test_, something that runs the application the way a real user would.

Instead of executing individual functions from specific systems, we're going to boot up **our entire game the same way a real player would**. We're going to click the "New Game" button on the main menu and test features where they lie within the real use case.

## Creating the right environment

If we want our integration test to work well, we have to rewrite our brains to _love_ crashing. **Fields of Mistria is eager to crash**. At the first whiff of a problem, we tear the entire thing down.

This, of course, has limits. Many of our crashes come from debug asserts, which will instead log a warning and proceed in production builds. But **error recovery in video games is extremely limited**. In some cases it might be easy to throw the error away, but we have to question the state we're left with afterwards.

A particular bug in v0.12.0 comes to mind:

{% <admonish kind="warning" title="Failure to spawn furniture object"> %}
The player is loading their save and we are placing all of their furniture objects on the map. One of those objects is somehow in an illegal position when we try to place it.
{% </admonish> %}

At the time, our solution to the above was to dismiss the error, ignore the object, and continue with our load. Preventing a user from loading their save seemed catastrophic, so surely it was better to just move past that one weird mistake and let life continue, no?

Soon after the release, we started to hear buzz that players' chests were missing from their farms. This sounded bizarre, as there's nothing special about chests in terms of how they save and load, so there's no reason that they _specifically_ would have a problem.

And they didn't! In actuality, _many_ users were having singular objects vanish. The difference is that chests are the most precious object for a player. A random rock wasn't where it was yesterday? You almost certainly wouldn't even notice. But your chest, containing all of your precious items? That's a huge issue -- that's worth a bug report.

{{ <figure src="chests.png" alt="A screenshot of Fields of Mistria where a player stands in front of many storage chests." caption="Players take their storage very seriously in Fields of Mistria." /> }}

This is our worst nightmare as far as bugs go (and is still ominously referred to as "the chest bug" at the studio today), because once we let them load in with that chest missing, **all it takes is saving over their file once for it to be gone forever**. If we can get a patch out soon enough, we can fix the problem and allow these objects to spawn again, but every second that ticks by, we know that more and more players are going to lose data.

Within the hour, the bug was fixed and a new build was being assembled for deployment. But what if we could go back in time, and instead, what if we had just crashed when we failed to load these objects?

The advantage isn't that it would have ultimately protected the users' save files until a patch was released (though that is true). The _real_ advantage is that _we_ would have noticed **days earlier when our integration tests failed**.

## Enter `TestSuite.gml`

`TestSuite.gml` is a file of roughly 6,000 lines of integration tests that eagerly seek out crashes. On our previous engine, the full set of tests could take up to seven hours to complete, so we had to run most of them in overnight cron jobs. On our new engine, we can run them headless with an uncapped time step. Split across 30 jobs, the entire thing typically finishes in **under 10 minutes**. That enables us to run them on every single push sent to GitHub, meaning we catch bugs as soon as they're created.

### Goals

For the test suite to do its job, a few things need to be true about it.

{% <steps> %}
1. **It's quick to work with.** Any engineer should be able to add a new test, or diagnose a failing one, without fuss.
2. **It's authentic, without constant upkeep.** It should test the game as realistically as possible, and spurious failures should be rare. No matter the policy, people start ignoring a suite once it has a reputation for crying wolf.
3. **It's aligned with the codebase.** The suite should fit the architecture of the project as a whole, so that its needs complement the needs of the code.
{% </steps> %}

### Example targets

These are three bugs loosely based on real incidents Fields of Mistria has encountered throughout its development. As we go through the suite, we're going to identify where and how they would be caught.

{% <admonish kind="bug" title="Bug #1: A missed rename"> %}
We rename a function that's called 40 times in the game's scripts. We update 39 of the calls, missing a single one that only triggers on 1 of the 100 possible floors in the mines.
{% </admonish> %}

{% <admonish kind="bug" title="Bug #2: A hard-coded value"> %}
We lower the heart level needed to invite an NPC to the Shooting Star Festival from 5 to 4. We update the quest details, dialogue, and tutorials, but miss that the number is hard-coded in the logic instead of being read from our data files. Despite what the game tells them, players are unable to progress their quest.
{% </admonish> %}

{% <admonish kind="bug" title="Bug #3: An ID typo"> %}
The Tomato Soup item has a typo in its internal ID: `tomato_soupp`. We fix the typo and update every reference to it in the codebase. A player has this item sitting on the ground on their farm, and after updating, their game crashes when they load their save.
{% </admonish> %}

### How tests work

Our tests are built on a core system in the codebase called `Chains`. If you've worked with coroutines, or asynchronous execution in general, the concept should feel familiar. A `Chain` is a series of `Link`s, each carrying some logic and a condition for moving on. Every tick, the chain runs through its links until one tells it to wait, then picks back up from there on the following frame.

{% <admonish kind="note"> %}
Fields of Mistria ships its scripts in plain text. We'll be simplifying the excerpts in this article to keep things focused, but if you own the game, you can read the real code behind everything we talk about. It's even possible to run the test suite on the shipped version of the game, but we'll leave figuring out how to the talented modding community.
{% </admonish> %}

```js,name=GML
var chain = new_chain(); // a new Chain, which the engine will update each tick
chain.append(LinkId.Timer, 5); // this timer takes 5 ticks to finish
chain.append(LinkId.Function, function() {
    trace("hello!"); // once the timer above is done, this prints
});
chain.append(LinkId.Await, function() {
    return some_condition(); // fires every tick until it returns true
});
```

{% <chain caption="Each link has to finish before the chain moves on to the next."> %}
{{ <chain_link name="Timer" detail="wait 5 ticks" /> }}
{{ <chain_link name="Function" detail='trace("hello!")' /> }}
{{ <chain_link name="Await" detail="until some_condition() is true" repeats={true} /> }}
{% </chain> %}

**The entire test suite is one very long chain.** As the suite is assembled, each test is handed that chain and is free to append its own links.

```js,name=TestSuite.gml
// Add a new test called `dungeons`, which starts a new run in the mines and
// goes through every single floor for one frame each, to make sure they can
// all load without error.
TS_TESTS.push({
    name: "dungeons",
    call: function(chain) {
        chain.append(LinkId.Function, function() {
            randomize();
            trace("Starting a dungeon run with random seed {}", random_get_seed());
            enter_dungeon(0, DUNGEON_FLOOR_COUNT);
        });

        chain.append(LinkId.Await, function() {
            return is_dungeon_room(room); // wait to actually enter the dungeon
        });

        repeat DUNGEON_FLOOR_COUNT {
            chain.append(LinkId.Function, DUNGEON_RUNNER.proceed);
            chain.append(LinkId.Timer, 1); // room swap occurs between frames
        }
    }
});
```

Note that there are no manual asserts or checks here. Again, the primary goal of the suite is just to find crashes. **It is _not_ a replacement for real, human QA**. We do write manual asserts at times, but later on we'll cover how we program our codebase to work naturally with this kind of testing.

{% <admonish kind="success" title="Bug #1: Caught"> %}
Our missed rename has been caught. As soon we open PR the suite runs, visits every floor, and crashes on the undefined call. Merging is blocked until it's fixed and the tests pass.
{% </admonish> %}

## Testing story content

Fields of Mistria has over 150 cutscenes. Players advance through dialogue at their own pace, but at an average one, **watching them all would take roughly 15 hours**. They're also highly specific, as there are twelve different characters to date. Whether a player is watching the shooting stars with their crush, asking them to dance at the Harvest Festival, or attending their wedding ceremony, the scenes and dialogue are different for each character. It's incredibly easy to introduce a bug in one scene with a seemingly innocuous edit somewhere else.

But crashes aren't the only thing we need to worry about; **softlocks are their own nightmare**. In many cases it'd almost be _better_ if they were crashes. A crash gets reported to [Sentry](https://sentry.io/welcome/)[^sentry] instantly with information about exactly where it happened and precisely how many players it's affecting. With a softlock, the player is just as stuck, but we have to wait on their reports and reproduction steps to diagnose it.

So our story tests serve two purposes. First, find any crashes hiding in the many cutscenes. Second, make sure all of the content is accessible exactly when we expect it to be.

```js,name=GML
// Test going to the Shooting Star Festival with Juniper. Elsie should visit us
// in the morning and give us the Brooch item, and then as long as we have at
// least 4 hearts with Juniper, we should be able to invite her. At 8pm we can
// go with her, and the scene should end the day.
SE.skip_to_date("summer 28");
SE.next(function() {
    // this is 4!
    var hearts_needed = todays_festival().date.minimum_hearts_required;
    NPCS[NpcId.Juniper].set_heart_level(hearts_needed);
});
SE.goto_scene_trigger("shooting_star_morning"); // Elsie visits us in the morning
SE.play_out_scene("shooting_star_morning"); // let the cutscene play
SE.goto(NpcId.Juniper); // teleport to Juniper

// If Elsie correctly gave us the Brooch item, it'll be in our hands, and the
// "invite" interaction should be available with Juniper since she's at the
// minimum 4 heart level.
SE.interact_with(obj_juniper, "misc_local/interact_invite");
SE.play_out_textbox(); // she says yes
SE.clock_jump("8:00pm"); // jump ahead to 8pm, when the festival starts
SE.goto(NpcId.Juniper); // teleport to her again

// If we succeeded in inviting her, we should be able to go to the festival
// with her now that it's past 8pm.
SE.interact_with(obj_juniper, "misc_local/go_to_festival");
SE.next(function() {
    // That interaction spawns the confirmation popup, so find it and tap the
    // "yes" button
    ANCHOR.tap_node(ANCHOR.get_menu(Menu.Popup).buttons.get(1));
});
SE.play_out_scene("shooting_star_juniper"); // Juniper's scene should be playing
SE.play_out_eod(); // the scene ends the day, so play out the end of day screen
```

`SE` is short for `StoryExecutor`, a set of tools for simulating gameplay authentically (within reason). We don't just call the internal code we're trying to exercise; we go through it from as high a level as we can, so we catch any issue the player could run into. For example, here's what `SE.end_day()` looks like:

```js,name=StoryExecutor.gml
function end_day(scene_to_expect) {
    // seeks out where the player's bed is in the world and moves us there
    self.goto("bed");
    self.next(function() {
        // Now we look for the actual bed in the room and interact with it. If
        // that works the way we expect, a popup confirming our choice should
        // have spawned. We then press the same "yes" button the player would.
        with obj_node_renderer {
            if self.node.prototype.bed_kind != undefined {
                interact(node);
                ANCHOR.tap_node(ANCHOR.get_menu(Menu.Popup).buttons.get(1));
            }
        }
    });

    // the sleep sequence is a mini cutscene, so we wait until it's done...
    self.play_out_scene("sleep");

    // ...then, if we expect a cutscene to play this evening, we have to say
    // so and play it out
    if scene_to_expect != undefined {
        self.play_out_scene(scene_to_expect);
    }

    // finally, we make sure we're on the end of day screen and hit the
    // "next day" button
    self.play_out_eod();
}
```

Each of these commands doesn't just perform the action it describes; **it also crashes if the game isn't in exactly the state that action implies**. If we tell it to play out a scene and that scene isn't playing, we crash. If we tell it to do something to progress a quest and that quest isn't active, we crash.

{% <admonish kind="success" title="Bug #2: Caught"> %}
Our hard-coded value has been caught. The test raises Juniper to the 4 hearts our data files ask for, but the logic is still secretly checking for 5, so the invite never becomes available. `SE.interact_with` fails to interact with her, and the suite crashes.
{% </admonish> %}

## Catching broken save files

Our project contains a folder called `upgrade_targets` full of save files, which come in two kinds:

{% <panels> %}
{% <panel title="Regression saves"> %}
One for every save-related bug we've ever had. They keep those bugs from coming back, and they're a good excuse to add more test candidates to the suite.
{% </panel> %}
{% <panel title="Stuffed saves"> %}
Every time we release a new version, a tool generates a save that aims to contain every possible piece of content: every piece of furniture placed, every single item in some chest, every unlockable present, and so on.
{% </panel> %}
{% </panels> %}

We're pretty consistent about adding patches whenever we PR an edit that affects saves, but this makes sure no mistake slips past. Every one of these saves gets loaded, and a basic set of tests is run over it to make sure nothing is corrupted.

{% <admonish kind="success" title="Bug #3: Caught"> %}
Soup crash caught! Deleting that single `p` will cause, at the very least, the most recent stuffed save to crash as we try to deserialize whichever chest we shoved the soup into.
{% </admonish> %}

This is how we've been able to guarantee that **a player's save file will always upgrade**, all the way back to our initial demo over two years ago.

## Writing code aligned with the suite

We've already discussed how we take a crash-forward approach on Fields of Mistria. That idea goes further with how we design our databases themselves, such that **they naturally demand testing**. For example, every quest is required to specify which test it expects to be completed in. When that test finishes, it looks at every quest that references it, and if any of them aren't in the player's completed quest log, the test fails. The same goes for other systems, like cutscenes and menus.

When a writer adds a new cutscene to the game, we _could_ ask them to always remember to get one of our engineers to update the tests, but it's far safer to expect the game to take care of itself. Every cutscene has to declare which test is responsible for it:

```toml,name=cutscenes.toml
[my_new_cutscene]
    # <...>
    # every cutscene must provide this field, otherwise we crash on launch
    test_target = "story_scenes"
```

If that test finishes without having played the new cutscene, the suite fails on the writer's PR with a clear error. It's less a failure than a useful reminder:

{% <terminal> %}
Test Suite Result: <span class="fail">FAILED</span>. The following cutscenes were not covered: my_new_cutscene.
{% </terminal> %}

## The results

After tens of thousands of suite runs, we can't really begin to estimate how much time and money this system has saved us. Honestly, it's **hard to imagine the game being possible without it**. When we decided to switch to our own engine built in Rust (which our Lead Programmer, [Jonathan Spira](https://github.com/sanbox-irl), covers in his RustConf talk, [_Oxidizing Fields of Mistria_](https://www.youtube.com/watch?v=OkzUNS0H2_g)), the test suite was the litmus test for whether it was ready for production. Short of shipping it, there was no bigger hurdle we could put in front of the new engine than getting through that enormous amount of execution. For months of its development, the suite doubled as a production tool, showing us exactly what was left to port over.

Plus, after over seven years in production, it's enormously reassuring to see this screen before hitting the big green release button on Steam.

{{ <figure src="tests_passed.png" alt="GitHub checks for the test suite, all passing" caption="yeah... yeah, that's the good stuff" /> }}


[^rust_loc]: Rust projects are obviously built on the shoulders of countless lines of open-source code, so this number is approximated from the Rust inside of our engine, [fabricator](https://github.com/NPC-Studio/fabricator) (our GameMaker runtime), and the Rust within Fields of Mistria itself.
[^team_members]: It's tough to measure a number like this due to a mix of full-time and part-time roles, people joining and leaving, and the external teams we worked with for things such as QA and localization. You can refer to [our MobyGames page](https://www.mobygames.com/game/229134/fields-of-mistria/) for the full credits of all the incredible folks who made the game possible.
[^sentry]: Sentry and error reporting deserve their own article, but needless to say, automatic crash reporting is essential. This isn't news to the software space, but many indie devs skip this. Don't!
