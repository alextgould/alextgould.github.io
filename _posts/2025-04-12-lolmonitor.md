---
layout: post
title: "LoL Monitor: helping players manage their gaming sessions"
date: 2025-04-12
description: "Do you want to \"three block\" your League of Legends gaming sessions, but struggle to maintain self control? This tool helps you by automatically closing the lobby for you."
img: lolmonitor/lolmonitor_wide_azure.jpg
tags: [Project] # Personal, Opinion, Technical, Review, Project, Testing
main_page_summary: "Building a tool to close the League of Legends lobby after each game and enforce breaks"
---

## Executive Summary

In this post I give an overview of the **LoL Monitor** application that I built to help League of Legends (LoL) players manage their gaming sessions by closing the lobby after each game and enforcing breaks between games.

This includes:
* An insight into why LoL is so engaging and addictive, and why players can benefit from using LoL Monitor.
* A dive into the Go programming language that was used to build the tool, contrasting it with the more commonly used Python language.
* A perspective on software development in the generative AI era, how tools such as Copilot can help developers create products faster, and why their limitations require an ongoing role of human developers.
* My personal experience using the tool, finding it to be surprisingly effective.

The code is available at [https://github.com/alextgould/lolmonitor](https://github.com/alextgould/lolmonitor).

## Table of Contents

- [Introduction](#introduction)
- [The ten-dimensional chess board](#the-ten-dimensional-chess-board)
- [Go next](#go-next)
- [Refactoring the refactored refactor](#refactoring-the-refactored-refactor)
- [Let's Go Go Go!](#lets-go-go-go)
- [Moving from prototype to polished product](#moving-from-prototype-to-polished-product)
- [It's kibble time!](#its-kibble-time)
- [Conclusion](#conclusion)

## Introduction

[League of Legends](https://en.wikipedia.org/wiki/League_of_Legends) (LoL) has been around for over 15 years and is currently the world's largest esport, with millions of concurrent players around the world. 

The game is face-paced and deceptively complex, requiring significant time and effort to achieve mastery and progress in the ranked system. It combines the social and team based aspects of traditional sports with the complexity and strategy of a ten-dimensional chess board with constantly evolving rules. While it can be fun and engaging, it can also be emotionally overwhelming, and can lead players to fall into psychological traps that result in them playing longer than they would like to.

Let's start with an insight into the complexity and emotional aspects of the game, as motivation for the importance of developing a tool to help players have a more healthy relationship with the game.

## The ten-dimensional chess board

At face value, LoL seems like a fairly simple game. At opposing corners of the map are two bases, out of which come waves of minions who march down three lanes to defeat the opposing force. Ten champions - five on each team - act to swing the tide of the battle. In general, champions only need to press four buttons to use their active abilities, along with moving and attacking. Champions get gold and/or experience for killing minions and securing objectives that appear over the course of the game. They can use the gold to buy items, and use the experience to improve their abilites, making them stronger as the game progresses. The premise is simple and it's easy for players to get started.

There are, however, many layers of complexity that make the game so engaging and popular from an esport perspective:

* **Champions** - As of 23 January 2025, there are 170 champions in LoL, of which 10 are chosen each game. In theory, this means there are over 4 quadrillion possible combinations that could appear. In reality the number is lower, as champions tend to take on specific roles in the game and some champions are more commonly picked than others, but there's still a staggering number of possible drafts, making each game unique from the start. New champions are introduced regularly, constantly adding new interactions to the mix.

* **Abilities** - Each champion has four active abilities and one passive ability. With 170 champions and 5 abilities each, that's around 850 abilities in total. The individual abilities are generally quite unique and can contain a lot of details, and knowing these details can decide the outcome of a skirmish. For example, here's the [wiki](https://wiki.leagueoflegends.com/en-us/Briar) entry for a *single* ability (Briar's W):

![]({{site.baseurl}}/assets/img/lolmonitor/briar_w.png)

* **Items** - Throughout the course of the game, players spend gold on items. There are over 200 different items in the game, which can significantly change the strength of champions and their abilities. This adds an additional layer of complexity as the game progresses. There have also been many significant changes to the items over the years, adding further complexity over the longer term.

* **Intuition** - The best players in the world will not only have a general knowledge of all of these abilities, but will have a fairly good sense of the *cooldowns*, *range* and *damage* potential of their opponent's abilities, punishing cooldowns by playing more aggressively during a short window, tethering their oppoenents by standing just outside the range of their abilites, baiting them into using their abilities and so on. These values can change as the game progresses (in the example above, Briar's cooldown reduces as she puts more points into the ability, and the damage potential will depend on defensive stats on items purchased by her opponents). The skill ceiling on these aspects is super high.

* **Mental stack** - There are many things to focus on and think about at every instant of the game - from the constantly changing health of minions and the micro movements and ability usage of you and your lane opponent, to the macro movements and strategy of your team (and the enemy team if your team has vision of them) on the map. As humans we have an inherently limited ability to process information - our *mental stack* becomes overloaded and we fail to gather or use the information we need to make the right decisions. Managing our inherent limitations in such an environment (for example through intentional practice or building intuition over many games) can be a complex process.

* **Humans are random** - Champions are piloted by humans, who are emotional creatures. They may be dealing with real life issues, feeling tired, stressed or angry, and this makes their in-game behaviour inherently unpredictable. Managing this aspect can be complicated, particularly for the majority of players who are playing in solo queue and may find it challenging to communicate effectively with their anonymous team mates in such a fast-paced game.

All of this makes for an engaging environment which can induce strong emotions. Every game is packed with small victories which trigger dopamine hits. The randomness inherent in the process is akin to a slot machine. Games are relatively short (~30 minutes) and easy to join (matchmaking with a huge player base, no real physical exertion required), so it's easy to rationalise playing "just one more". 

Which brings us to the post game lobby window...

## Go next

There are two options coming out of a game of League of Legends, and both are bad news for playing in moderation.

Option 1: Victory! You're on a dopamine high, you're feeling great, and there's a button asking if you want some more. Of course I want more! Let's play another game!

![]({{site.baseurl}}/assets/img/lolmonitor/victory.png)

(If we pause for a moment, we can see that there is indeed a little X that would let us escape the infinite loop. It's hidden to the left of the giant glowing CONTINUE button.)

Option 2: Defeat! You played poorly and you're feeling shame, guilt, frustration. Or you played well, but you lost anyway; your teammates were underperforming, your team ignored your call and hung you out to dry, your team fought at a numbers disadvantage while you weren't there. It feels unfair and you're feeling frustrated. Either way, the easiest way to make these negative feelings go away is to sweep it under the rug and pretend it never happened. Next game you'll be better, or your team will perform better. Let's play another game!

![]({{site.baseurl}}/assets/img/lolmonitor/defeat_demoted.png)

In both cases, the main problem is that there's a big button that continues the endless cycle. It's like doom scrolling on social media or autoplay on YouTube. You need something to help you break the cycle. You need a moment so you can pause, reset and make a more logical decision rather than making an emotional one. You need that button to go away.

## Refactoring the refactored refactor

The premise of the idea is simple. If we can close the lobby window at the end of each game, the continue button will go away. If we can get that basic functionality going, the rest of the program logic should be fairly straightforward.

With no background in Go, using the magic of ChatGPT and Copilot to hit the ground running, I was able to get a basic prototype up and running in less than a day. What ensued was several days of refactoring, essentially rewriting all the code that ChatGPT gave me and restructuring my project a few times. I took the scenic route to better understand the basics of the Go language, but also to experiment with some "best practices". Hence, before we dive into some actual Go code, I want to first talk about how the project folder is strutured, as this gives some insights into the world of Go.

The folders containing .go files in my repo are as follows:

```bash
lolmonitor/       
├── cmd/                        
│   └── lolmonitor/
│       └── main.go             # Entry point for the application
├── internal/                   # Internal packages that are specific to this application
│   ├── application/
│   │   ├── orchestrate.go      # The main loop, monitors for windows and takes actions
│   │   └── orchestrate_test.go 
│   ├── config/                 
│   │   ├── config.go           # Handles interactions with the config.json configuration file
│   │   └── config_test.go      
│   ├── infrastructure/         
│   │   └── startup/
│   │       ├── startup.go      # Uses Windows Registry to auto-load at startup
│   │       └── startup_test.go 
│   └── interfaces/             
│       └── notifications/
│           ├── toast.go        # Displays windows toast notifications
│           └── toast_test.go   
└── pkg/                        # Public package that can be imported by other modules
    └── window/                 
        ├── close.go            # Forcibly closes windows
        ├── close_test.go       
        ├── monitor.go          # Monitors for the opening and closing of windows
        ├── monitor_test.go     
        └── wmi.go              # Windows Management Instrumentation (WMI) Functions and Types
```

There are a few features in this structure that explain why I took the scenic route in turning my initial prototype into a more polished product:

1. The **pkg** folder contains a general purpose "window" package that can be used to monitor for the opening and closing of windows by name, as well as forcibly shut them. While this code could have comfortably lived with the main application code, I refactored and split it out as it felt like it could be a useful tool for other projects. This is used to highlight a design feature of Go "modules" (i.e. the project level), which contain many "packages" (i.e. the directory level): the location of the package actually determines whether it can be imported by other modules or not. Packages within the internal directory *can't* be imported externally, whereas those in the pkg package directory can.

    This highlights a general characteristic of the Go language compared to Python, which is that it is much more opinionated and structured. It will add, remove and rearrange included packages, and adjust indenting, when you save the file. It won't let you import a function into another package unless it starts with an upper case letter to indicate that it's an external function. It also checks for compile errors, so you can't actually run the code until it decides it's ready to be run.

2. The **internal** folder uses a series of subfolders to achieve "separation of concerns", where we have different packages for each purpose and/or dependency, which are then brought together by a higher level package. This is a common design feature in Go projects, and while the Go community have established [quite a few](https://medium.com/@smart_byte_labs/organize-like-a-pro-a-simple-guide-to-go-project-folder-structures-e85e9c1769c2) different "best practice" structures, these differ mainly in how they choose to structure their internal folder. Such project structures can make it easier to find and update specific sections of the code base and let you more easily "plug and play" different components (e.g. swap one database for another).

    While it was probably overkill for such a small project, I refactored the code to have four separate packages in the internal folder:
      * **application** - runs the main loop, monitoring for windows using the windows package from the *pkg* folder and applying the domain "logic" of when to close the lobby (in bigger projects a separate *domain* folder might also be split out for this logic)
      * **startup** - deals with the windows registry; this sits in the *infrastructure* folder, which typically contains packages that relate to technology-specific tools that the application uses, such as databases, file systems, operating system or external APIs
      * **notifications** - handles the windows toast notifications; this sits in the *interfaces* folder, which typically contains packages that allows users (or systems) to interact with the application
      * **config** - creates and loads the config.json file; this could also have been placed within the *infrastructure* folder, but it's also commonly split out

    In Python, it's common to have more code within a single .py file and/or place all the source code into a single /src folder. The Go style feels more verbose, particularly for a small module like this one, but I can already feel the benefits of being able to feel confident in the various areas within the codebase. The Go style feels much more scalable and robust.

3. For almost every .go file there is a corresponding **_test.go** file. In contrast to Python, where error handling and unit tests are "best practice" but are often more of an afterthought, in Go these aspects are first-class citizens. Almost every function will return an error value to be handled by the function that calls it. The code, which has been segmented into distinct packages, will be tested in isolation, so that when a package is used by a higher level package, you can be fairly confident it will all work. There's a built in testing package with conventions such as having corresponding _test.go files which contain a series of functions that start with Test and can be run (at least in vscode) with a button click, in isolation or as a batch.

    In the case of the *windows* package, creating the test files actually involved a decent amount of refactoring. Initially, my test files required me to manually open and close windows, which was a slow process and would have been a pain when testing time of day exclusion periods. In order to test this properly, I had to adjust the code to use an *Interface*, which could be set to either the real process retriever (which looked for actual processes running in the Windows operating system) or a Mock[^1] process retriever (which simulated processes being opened and closed). Similarly, functions that use the current time in production can be written to use a parameterised time value, which is set to the current time in production but can be artificially set in the test environment.

	While this took some additional time, both in getting used to the way testing works and to actually implement tests, I can see how this approach can lead to much greater confidence in your code. I found it useful to have Copilot create some sensible tests, which I could then adjust as needed. Creating the tests is often relatively straightforward but can be a little tedious, so it's nice to outsource it and have it done immediately. It's also good to have multiple eyes involved, as sometimes Copilot would create tests that I wouldn't have and vice versa. I think with more experience, it would also take less time as you would know to use interfaces and parameterised values in advance, making it much easier to write the tests.

With our project well structured, we can now look at some of the individual Go files that make the magic happen.

## Let's Go Go Go!

### WMI

To monitor for windows opening and closing, we can use the Windows Management Instrumentation (WMI) infrastructure. This is essentially a table which we can query using Windows Query Language (WQL), which is effectively SQL.

The basic code used to query the WMI table to see if a process exists looks like this:

```Go
import (
	"github.com/StackExchange/wmi"
)

// Win32_Process represents a Windows process
type Win32_Process struct {
	Name      string
	ProcessId uint32
}

// ProcessRetriever is an interface to allow mocking WMI process sourcing
type ProcessRetriever interface {
	GetProcessesByName(name string) ([]Win32_Process, error)
}

// ProcessRetriever will be WMIProcessRetriever when live
type WMIProcessRetriever struct{}

func (w WMIProcessRetriever) GetProcessesByName(name string) ([]Win32_Process, error) {
	var processes []Win32_Process
	query := fmt.Sprintf("SELECT Name, ProcessId FROM Win32_Process WHERE Name = '%s'", name)

	maxRetries := 5
	retryDelay := 5 * time.Second

	for i := 0; i < maxRetries; i++ {
		err := wmi.Query(query, &processes)
		if err == nil {
			return processes, nil
		}
		log.Printf("WMI query failed (attempt %d/%d): %v", i+1, maxRetries, err)
		time.Sleep(retryDelay)
	}

	return nil, fmt.Errorf("WMI query failed after %d retries", maxRetries)
}
```

Thankfully ChatGPT has memorised the internet and can quickly point me in the direction of the wmi package, so we can build on the shoulders of giants. There's a single line here that does most of the work, using wmi.Query to run the SELECT query to see if the process exists. The rest of the code deals with some of the aspects discussed in the previous section, creating the ProcessRetriever interface with a function to get processes by name, which is set to the WMIProcessRetriever in production. The _test.go equivalent of this looks like this:

```Go
type MockProcessRetriever struct {
	Processes map[string]Win32_Process
}

func (m *MockProcessRetriever) GetProcessesByName(name string) ([]Win32_Process, error) {
	var result []Win32_Process
	for _, process := range m.Processes {
		if process.Name == name {
			result = append(result, process)
		}
	}
	return result, nil
}

func (m *MockProcessRetriever) AddProcess(name string, pid uint32) {
	m.Processes[name] = Win32_Process{Name: name, ProcessId: pid}
}

func (m *MockProcessRetriever) RemoveProcess(name string) {
	delete(m.Processes, name)
}
```

Similar to the ProcessRetriever, the MockProcessRetriever also has a GetProcesssesByName function, but it has the additional AddProcess and RemoveProcess functions so we can control the processes in the []Win32_Process *slice* (the Go equivalent of a Python list).

This is also our first look at a function that returns error type values, along with the `if err == nil` style checking, which we'll be seeing a lot of as we go along.

We can use the GetProcessesByName function to check if a process is active or not, and this can be used in a simple loop to identify when new processes appear, or existing processes disappear:

```Go
type ProcessEvent struct {
	Name      string
	PID       uint32
	Type      string // open, close
	Timestamp time.Time
}

func MonitorProcess(name string, events chan<- ProcessEvent, delaySeconds int, pr ProcessRetriever) {

	wasActive := false
	for {
		isActive, id, _ := isProcessActive(name, pr)
		if !wasActive && isActive {
			events <- ProcessEvent{Name: name, PID: id, Type: "open", Timestamp: time.Now()}
		} else if wasActive && !isActive {
			events <- ProcessEvent{Name: name, PID: id, Type: "close", Timestamp: time.Now()}
		}
		time.Sleep(time.Duration(delaySeconds) * time.Second)
		wasActive = isActive
	}
}
```

In the above function, we use a *channel* to capture the event of a process opening or closing. We first define the ProcessEvent struct to hold our events. A struct is similar to a Python class, in that it bundles related data together and can have methods, differs in that it lacks the inheritence, *self* and *\_\_init\_\_* features. When there's a change in the results of the WMI SELECT query, we capture this as an event and send it to the *events* channel. This channel will be created by the package that kicks off the MonitorProcess function, which will also contain an infinite loop to process events as they arise.

The last aspect of the windows package is the task killer. Our events contain the process id, which we can use with the taskkill command to close a window:

```Go
func (w WMIProcessKiller) KillProcess(pid uint32) error {
	cmd := exec.Command("taskkill", "/PID", fmt.Sprint(pid), "/F")
	return cmd.Run()
}
```

As with the ProcessRetriever, this is really just a single line of code, with a lot of extra code to create an interface and a mock version for testing, handling of errors and so on.

With our window package ready, we can start using it in our main application.

### Main application

The main application is split between main.go and orchestrate.go.

The main.go file is the main entry point for the application and is designed to be very basic and not require any testing, essentially just calling the relevant functions from the config, startup and application packages in turn, as well as keeping our event channel open:

```Go
func main() {
	// Load (or create) config file
	cfg, err := config.LoadConfig("")
	if err != nil {
		log.Panicf("Failed to load config: %v", err)
	}

	// Add or remove startup process based on config settings
	if cfg.LoadOnStartup {
		err = startup.ConfirmLoadOnStartup()
	} else {
		err = startup.ConfirmNoLoadOnStartup()
	}
	if err != nil {
		log.Panicf("Error updating startup: %v", err)
	}

	// Create channel for window events
	gameEvents := make(chan window.ProcessEvent, 10) // Buffered channel to prevent blocking

	// Start monitoring in a separate goroutine
	go application.Monitor(cfg, gameEvents, nil, nil)

	// Handle Ctrl+C for graceful shutdown
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, os.Interrupt)

	<-signalChan // Block until an interrupt signal is received (or else indefinitely)
	close(gameEvents) // Signal monitoring to stop
}
```

There are however a few interesting new things here:

1. We use `make` to create the gameEvents channel that will collect events from our WMI monitoring process. The second parameter in the make function is the buffer size of the channel, which allows for multiple unprocessed events to exist in the channel at once. In normal operation, we probably won't see 10 events between checks, but this figure gives us a safety margin, which might be useful for example when testing.

    Now is probably a reasonable time to note that Go uses static typing, but := allows you to create variables as you go, so long as you use = the next time the variable value is set.

2. We use the `go` keyword when calling application.Monitor to create a goroutine, which is a lightweight function that can run concurrently with other functions. This is one of the key features and strengths of the Go language - it has built-in concurrency which allows Go programs to perform many tasks simultaneously without much overhead. This contrasts with Python, where external libraries such as asyncio are needed to provide an event loop for managing asynchronous tasks, which is more complex and involves more overhead.

    In this case, it lets us run the application.Monitor function while continuing to run the main() function. The main() function creates a channel called signalChan that monitors for the interupt signal that occurs when you press "Ctrl+C" in a terminal. So long as nothing is added to the channel, the function will simply wait at the <- point. This means the gameEvents channel which was created earlier will remain open indefinitely.
	
The orchestrate.go file contains the main loop which checks for windows opening and closing and applies the logic of whether to force close the lobby window. The file is actually somewhat large, so for the purpose of this blog I've removed some elements (constant values, default ProcessRetriever and ProcessKiller values, some comments and logging, live updating of config values if the file changes etc).

The main function, Monitor, will use two helper functions. The first helper function encapsulates the logic of determining whether we're allowed to open the lobby or not:

```Go
func isLobbyBan(cfg config.Config, currentTime, endOfBreak time.Time) (bool, error) {

	// check we are not the endOfBreak exclusion period
	if currentTime.Before(endOfBreak) {
		return true, nil
	}

	// check if we are not outside of the [DailyStartTime, DailyEndTime] allowed period
	currentHM := currentTime.Format("15:04")
	if (cfg.DailyStartTime != "00:00" && cfg.DailyStartTime != "" && currentHM <= cfg.DailyStartTime) || (cfg.DailyEndTime != "00:00" && cfg.DailyEndTime != "" && currentHM >= cfg.DailyEndTime) {
		return true, nil
	}

	// otherwise the Lobby is not banned
	return false, nil
}
```
By splitting this out, it's easy to test, passing dummy config values and a currentTime value which isn't actually the real current time. Note when comparing time values, `currentTime.Before(endOfBreak)` is a Go way of doing a currentTime < endOfBreak comparison using time.Time values, whereas currentHM >= cfg.DailyEndTime is actually doing string comparison on the times formatted as "hh:ss".

The second helper function applies the logic .... UP TO HERE

```Go
// post game logic to determine the new sessionGames and endOfBreak values
func postGame(cfg config.Config, currentTime, gameStartTime, gameEndTime time.Time, sessionGames int) (int, time.Time, time.Duration) {
	var gameDuration, breakDuration time.Duration

	gameDuration = gameEndTime.Sub(gameStartTime)

	// increment game session count
	if gameDuration < time.Duration(cfg.MinimumGameDurationMinutes)*time.Minute {
		log.Println("Game was below the minimum duration (remake).")
		return sessionGames, currentTime, 0
	}
	sessionGames++

	// update required break duration based on config settings
	if sessionGames >= cfg.GamesPerSession && cfg.GamesPerSession != 0 { // use 0 to disable GamesPerSession functionality
		sessionGames = 0
		breakDuration = time.Duration(cfg.BreakBetweenSessionsMinutes) * time.Minute
		log.Printf("Enforcing a session ban of %v minutes", cfg.BreakBetweenSessionsMinutes)
	} else {
		breakDuration = time.Duration(cfg.BreakBetweenGamesMinutes) * time.Minute
		log.Printf("Enforcing a game ban of %v minutes", cfg.BreakBetweenGamesMinutes)
	}
	endOfBreak := gameEndTime.Add(breakDuration)
	return sessionGames, endOfBreak, breakDuration
}
```



```Go
func Monitor(cfg config.Config, events chan window.ProcessEvent, pr window.ProcessRetriever, pk window.ProcessKiller) {

	go window.MonitorProcess(LOBBY_WINDOW_NAME, events, CHECK_FREQUENCY_SECONDS, pr)
	go window.MonitorProcess(GAME_WINDOW_NAME, events, CHECK_FREQUENCY_SECONDS, pr)

	var gameStartTime, gameEndTime, endOfBreak time.Time
	var breakDuration time.Duration
	var sessionGames int
	lastChecked := time.Now()

	for event := range events {

		// reset session games if there was a significant gap since the last game ended (and we're either opening the lobby or starting a new game)
		if sessionGames > 0 && event.Type == "open" && gameEndTime.Add(time.Duration(cfg.BreakBetweenSessionsMinutes)*time.Minute).Before(time.Now()) {
			sessionGames = 0
		}

		if event.Name == GAME_WINDOW_NAME {
			if event.Type == "open" {
				log.Println("Game started")
				gameStartTime = event.Timestamp
				window.WaitForProcessClose(GAME_WINDOW_NAME, 1, pr) // check frequently
			} else {
				log.Println("Game ended")
				gameEndTime = event.Timestamp
				sessionGames, endOfBreak, breakDuration = postGame(cfg, time.Now(), gameStartTime, gameEndTime, sessionGames)

				// always close the lobby after a game, unless breakDuration is 0
				if breakDuration > 0 {

					// optional delay until close takes effect
					if cfg.LobbyCloseDelaySeconds > 0 {
						notifications.DelayClose(cfg.LobbyCloseDelaySeconds, sessionGames, cfg.GamesPerSession)
						time.Sleep(time.Duration(cfg.LobbyCloseDelaySeconds) * time.Second)
					}

					log.Printf("Closing the lobby. Break will expire at: %v", endOfBreak.Format("2006/01/02 15:04:05"))
					err := window.Close(LOBBY_WINDOW_NAME, pr, pk)
					if err != nil {
						log.Printf("Error: %v", err)
					}
					notifications.EndOfGame(endOfBreak, sessionGames, cfg.GamesPerSession)
				}
			}
		} else if event.Type == "open" {
			log.Println("Lobby was opened")

			// check if Lobby is banned and force close if so
			isLobbyBan, err := isLobbyBan(cfg, time.Now(), endOfBreak)
			if err != nil {
				log.Printf("Error: %v", err)
			} else if isLobbyBan {
				log.Println("Closing the lobby, which was opened during an enforced break.")
				err := window.Close(LOBBY_WINDOW_NAME, pr, pk)
				if err != nil {
					log.Printf("Error: %v", err)
				}
				notifications.LobbyBlocked(endOfBreak)
			}
		}
	}
}
```










## Moving from prototype to polished product

### Default settings

When you run the program, it will:
1. Close the Lobby after each game. There will be a 30 second delay, so you have time to honor your team mates or take a quick look at the damage chart.
2. Allow you to play 3 games in a session (a ["3 block"](https://www.youtube.com/watch?v=6K1xBJCjxi0&ab_channel=BrokenByConcept)) with a short 5 minute break between each game.
3. Ignore games under 15 minutes in duration (remakes).
4. Require a longer break of 1 hour between sessions.
5. Load automatically whenever you log into windows.

## Customise your experience

You can change things to suit your preferences, by editing the values in the config.json file, using any sort of text editor, such as Notepad. The table below shows the values available in the file:

| Parameter                     | Description                                                                 | Default Value | Alternative Values | Value to Disable |
|------------------------------|-----------------------------------------------------------------------------|---------------|----------------------------|------------------|
| breakBetweenGamesMinutes     | Minimum enforced break (in minutes) between individual games.              | 5             | 1, 15                     | 0                |
| breakBetweenSessionsMinutes  | Minimum enforced break (in minutes) between gaming sessions.               | 60            | 30, 120                     | 0                |
| gamesPerSession              | Maximum number of games allowed in a single session.                       | 3             | 1, 2                       | 0                |
| minimumGameDurationMinutes   | Minimum duration (in minutes) a game must last to count towards a session. | 15            | 5, 20                     | 0                |
| lobbyCloseDelaySeconds       | Delay (in seconds) before closing the lobby window.             | 30            | 0, 60                     | 0                |
| dailyStartTime               | Earliest time of the day when you can open the lobby (24-hour format).          | 00:00         | 04:00, 06:00               | 00:00            |
| dailyEndTime                 | Latest time of the day when you can open the lobby (24-hour format).                    | 00:00         | 22:00, 23:00               | 00:00            |
| loadOnStartup                | Whether the program should auto-load when you log in.                    | true          | false                      | n/a              |

A few examples:

* Let's say you want to only play one game each day, so the game is more memorable and you have time to review it in detail and reflect about it. You would adjust gamesPerSession to 1.

* Let's say you are happy to play as many games as you like (particularly on the weekend), so long as you don't queue after 10pm, because you don't like being overtired - it makes you play worse the next day. You disable the session games limit by setting gamesPerSession to 0 (and may also disable breakBetweenGamesMinutes so you can play continuously) and set the dailyEndTime value to "22:00".

* I personally can't decide whether it's good to have a delay before the lobby is closed. I like the idea of honoring my team, but I feel there's some residual risk of questionable behaviour, such as checking profiles to see if someone was a smurf or reading the post game chat where someone might make a toxic comment about you. I also suspect there's quite a few people out there who would rather not see their rank, to help reduce *ranked anxiety*. If any of this resonates with you, you can set the lobbyCloseDelaySeconds to 0, so the lobby closes instantly.

## Frequently Asked Questions

* What if something goes wrong? I'm 30 minutes into a game and my internet cuts out. What will I do then?

If anything goes wrong, you can always just open Task Manager (right click the task bar at the bottom of the screen and it's towards the bottom of the context menu that appears), find lolmonitor.exe in the list and right click > End task.

* Now that I know I can close down lolmonitor.exe (or edit the config.json file), won't I just cheat and do this to keep playing?

You're welcome to do this! However, you may find that having the lobby close automatically after each game is surprisingly effective. The main goal is just to get the emotional autopilot monkey brain out of the way. If you find that the post game lobby environment makes you want to keep playing longer than you should, try setting the lobbyCloseDelaySeconds to 0 so you never have to deal with it.

* What if I don't trust you?

This project is open source, so you're free to look through the code - or ask ChatGPT to do so for you - to gain some comfort. Then you can compile the code into an .exe file yourself using the instructions at the top of [main.go](/cmd/lolmonitor/main.go). If you haven't already, you'll need to have some sort of IDE such as [vscode](https://code.visualstudio.com/) and [install Go](https://go.dev/dl/). You may also want to change loadOnStartup to false in the config.json file, and you can run Registry Editor to confirm it's been uninstalled.

* That's a cool logo?

Thanks! It was made using Generative AI and any resemblances to the actual League of Legends logo are purely coincidental 😛. The three golden squares symbolise the ["3 block"](https://www.youtube.com/watch?v=6K1xBJCjxi0&ab_channel=BrokenByConcept) process, and how this helps contain the League of Legends gaming experience.






## It's kibble time!

Given my personal interesting in the subject matter, I decided it was only appropriate I did a bit of [dogfooding](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) before I wrote this blog post. This served to iron out a few wrinkles in the product and improve its design.

Before firing it up for the first time, I felt a mixure of fear and skepticism, which is funny given I'm the developer, but it's testament to the emotional side of playing LoL. I was worried it wouldn't work; that I would just override the tool and play another game and the whole thing would be a waste of time.

The first evening I went 0-3 (i.e. I lost every game) with an A rating or higher in every game, so there's a decent chance it was my team letting me down, which can be frustrating. What was surprising was how I felt when the tool kicked in and told me I had 10 seconds to look at the lobby and then had to take a break. I felt *relieved*. I was suddenly more aware of how I was feeling both physically and emotionally. After the second game, I was actually looking forward to finishing my third game and being kicked off - win or loss - because I knew it was an opportunity to go play a Cosy game or do a bit of coding. It was surprisingly effective.

TODO: update towards the end of the week

## Conclusion

Wrap up and reinforce the main takeaway (probably Thai), key points, next steps, call to action etc.[^1]

Future development - Mac?

---
<small>Footnotes:</small>

[^1]: Interestingly, the term Mock seems to be used loosely in the Go community. Technically a Mock is a configurable double that fails the test if its expectations aren't met. Often what is being referred to is actually a Fake (e.g. in-memory database instead of a real database), a Stub (returns pre-defined data) or a Spy (records calls). In my case, the window interface is actually a Fake.