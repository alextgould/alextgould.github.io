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
* My personal experience using the tool, finding it to be surprisingly effective.

The full code is available at [https://github.com/alextgould/lolmonitor](https://github.com/alextgould/lolmonitor).

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

At face value, LoL seems like a fairly simple game. At opposing corners of the map are two bases, out of which come waves of minions who march down three lanes to defeat the opposing force. Ten champions - five on each team - act to swing the tide of the battle. Champions generally have four active abilities to use, along with moving, basic attacks and a passive ability. Champions get gold and/or experience for killing minions and securing objectives that appear over the course of the game. They can use the gold to buy items, and use the experience to improve their abilites, making them stronger as the game progresses. The premise is simple and it's easy for players to get started.

There are, however, many layers of complexity that make the game so engaging and popular from an esport perspective:

* **Champions** - As of 23 January 2025, there are 170 champions in LoL, of which 10 are chosen each game. In theory, this means there are over 4 quadrillion possible combinations that could appear. In reality the number is lower, as champions tend to take on specific roles in the game and some champions are more commonly picked than others, but there's still a staggering number of possible drafts, making each game unique from the start. New champions are introduced regularly, constantly adding new interactions to the mix.

* **Abilities** - Each champion has four active abilities and one passive ability. With 170 champions and 5 abilities each, that's around 850 abilities in total. The individual abilities are generally quite unique and can contain a lot of details, and knowing these details can decide the outcome of a skirmish. For example, here's the [wiki](https://wiki.leagueoflegends.com/en-us/Briar) entry for a *single* ability (Briar's W):

![]({{site.baseurl}}/assets/img/lolmonitor/briar_w.png)

* **Items** - Throughout the course of the game, players spend gold on items. There are over 200 different items in the game, which can significantly change the strength of champions and their abilities. This adds an additional layer of complexity as the game progresses. There have also been many significant changes to the items over the years, adding further complexity across time.

* **Intuition** - The best players in the world will not only have a general knowledge of all of these abilities, but will have a fairly good sense of the *cooldowns*, *range* and *damage* potential of their opponent's abilities, punishing cooldowns by playing more aggressively during a short window, tethering their opponents by standing just outside the range of their abilites, baiting them into using their abilities and so on. These values can change as the game progresses (in the example above, Briar's cooldown reduces as she puts more points into the ability, and the damage potential will depend on defensive stats on items purchased by her opponents). The skill ceiling on these aspects is super high.

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
│   ├── interfaces/             
│   │   └── notifications/
│   │       ├── toast.go        # Displays windows toast notifications
│   │       └── toast_test.go   
│   └── utils/             
│       └── path.go             
└── pkg/                        # Public package that can be imported by other modules
    └── window/                 
        ├── close.go            # Forcibly closes windows
        ├── close_test.go       
        ├── monitor.go          # Monitors for the opening and closing of windows
        ├── monitor_test.go     
        └── wmi.go              # Windows Management Instrumentation (WMI) Functions and Types
```

There are a few features in this structure that explain why I took the scenic route in turning my initial prototype into a more polished product:

1. The **pkg** folder contains a general purpose "window" package that can be used to monitor for the opening and closing of windows by name, as well as forcibly shut them. While this code could have comfortably lived with the main application code, I refactored and split it out as it felt like it could be a useful tool for other projects. This is used to highlight a design feature of Go "modules" (i.e. the project level) and their "packages" (i.e. the subfolder level): the location of the package actually determines whether it can be imported by other modules or not. Packages within the internal directory *can't* be imported externally, whereas those in the pkg package directory can.

    This highlights a general characteristic of the Go language compared to Python: it's more opinionated. For example, when you save the file it will make changes to your code, such as adjusting the included packages to reflect what's actually being used and adjusting indenting based on its preferred style. Go treats functions that start with an upper case letter as external ones, preventing you from importing internal functions that start with lower case letters into other packages. Go also checks for compile errors, so you can't actually run the code until it decides it's ready to be run. Once you get used to it, this more structured approach is actually quite helpful.

2. The **internal** folder uses a series of subfolders to achieve "separation of concerns", where we have different packages for each purpose and/or dependency, which are then brought together by a higher level package. This is a common design feature in Go projects, and while the Go community have established [quite a few](https://medium.com/@smart_byte_labs/organize-like-a-pro-a-simple-guide-to-go-project-folder-structures-e85e9c1769c2) different "best practice" structures, these differ mainly in how they choose to structure their internal folder. Such project structures can make it easier to find and update specific sections of the code base and let you more easily "plug and play" different components (e.g. swap one database for another).

    While it was probably overkill for such a small project, I refactored the code to have several separate smaller packages in the internal folder:
      * **application** - runs the main loop, monitoring for windows using the windows package from the *pkg* folder and applying the domain "logic" of when to close the lobby (in bigger projects a separate *domain* folder might also be split out for this logic)
      * **startup** - deals with the windows registry; this sits in the *infrastructure* folder, which typically contains packages that relate to technology-specific tools that the application uses, such as databases, file systems, operating system or external APIs
      * **notifications** - handles the windows toast notifications; this sits in the *interfaces* folder, which typically contains packages that allows users (or systems) to interact with the application
      * **config** - creates and loads the config.json file; this could also have been placed within the *infrastructure* folder, but it's also commonly split out.
	  * **utils** - contains general purpose functions used by multiple other packages; this could alternatively live within the *pkg* folder for use in other modules.

    In Python, it's common to have more code within a single .py file and/or place all the source code into a single /src folder. The Go style feels more verbose, particularly for a small module like this one, but I can already feel the benefits of being able to feel confident in the various areas within the codebase. The Go style feels much more scalable and robust.

3. For almost every .go file there is a corresponding **_test.go** file. In contrast to Python, where error handling and unit tests are "best practice" but are often more of an afterthought, in Go these aspects are first-class citizens. Almost every function will return an error value to be handled by the function that calls it. The code, which has been segmented into distinct packages, will be tested in isolation, so that when a package is used by a higher level package, you can be fairly confident it will all work. There's a built in testing package with conventions such as having corresponding _test.go files which contain a series of functions that start with "Test" and can be run (at least in vscode) with a button click, in isolation or as a batch.

    In the case of the *windows* package, creating the test files actually involved a decent amount of refactoring in itself. Initially, my test files required me to manually open and close windows, which was a slow process and would have been a pain when testing time of day exclusion periods. In order to test this properly, I had to adjust the code to use an *Interface*, which could be set to use either a real process retriever (which looked for actual processes running in the Windows operating system) or a Mock[^1] process retriever (which simulated processes being opened and closed). Similarly, functions that use the current time in production can be written to use a parameterised time value, which is set to the current time in production but can be artificially set in the test environment.

	While this took some additional time, both in getting used to the way testing works and to actually implement tests, I can see how this approach can lead to much greater confidence in your code. I found it useful to have Copilot create some sensible tests, which I could then adjust as needed. Creating the tests is often relatively straightforward but can be a little tedious, so it's nice to outsource it and have it done immediately. It's also good to have multiple eyes involved, as sometimes Copilot would create tests that I wouldn't have and vice versa. I think with more experience, it would also take less time as you would know to use interfaces and parameterised values in advance, meaning there would be less refactoring involved and tests would be easier to create.

With our project well structured, we can now look at some of the individual Go files that make the magic happen.

## Let's Go Go Go!

### WMI... the Whale-sized Marshmallow Initiative?

To monitor for windows opening and closing, we can use the Windows Management Instrumentation (WMI) infrastructure. This is essentially a table which we can query using Windows Query Language (WQL), which is effectively SQL.

The basic code used to query the WMI table to see if a process exists looks like this:

```go
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

Thankfully ChatGPT has memorised the internet and can quickly point me in the direction of the wmi package, so we can build on the shoulders of giants. There's a single line here that does most of the work, using the wmi.Query function to run the SELECT query to see if the process exists. This function updates a []Win32_Process *slice* (the Go equivalent of a Python list), which is passed by reference using the & prefix. The rest of the code deals with aspects discussed in the previous section, creating the ProcessRetriever interface with a function to get processes by name, which is set to the WMIProcessRetriever in production. The _test.go equivalent looks like this:

```go
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

Similar to the ProcessRetriever, the MockProcessRetriever also has a GetProcesssesByName function, but it also has the AddProcess and RemoveProcess functions so we can manually control the processes in the slice.

We can use the GetProcessesByName function to check if a process is active or not, and this can be used in a simple loop to identify when new processes appear, or existing processes disappear:

```go
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

In the above function, we use a *channel* to capture the event of a process opening or closing. We first define the ProcessEvent struct to hold our events. A struct is similar to a Python class, in that it bundles related data together and can have methods, but differs in that it lacks the inheritence, *self* and *\_\_init\_\_* features. When there's a change in the results of the WMI SELECT query, we capture this as an event and send it to the *events* channel. This channel will be created by the package that kicks off the MonitorProcess function, which will also contain an infinite loop to process events as they arise.

The last aspect of the windows package is the task killer. Our events contain the process id, which we can use with the taskkill command to close a window:

```go
func (w WMIProcessKiller) KillProcess(pid uint32) error {
	cmd := exec.Command("taskkill", "/PID", fmt.Sprint(pid), "/F")
	return cmd.Run()
}
```

With our window package ready, we can start using it in our main application.

### Main application

The main application is split between main.go and orchestrate.go.

The main.go file is the main entry point for the application and is designed to be very basic and not require any testing, essentially just calling the relevant functions from the config, startup and application packages in turn, as well as keeping our event channel open:

```go
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

There are two interesting new things here:

1. We use `make` to create the gameEvents channel that will collect events from our WMI monitoring process. The second parameter in the make function is the buffer size of the channel, which allows for multiple unprocessed events to exist in the channel at once. In normal operation, we probably won't see 10 events between checks, but this figure gives us a safety margin, which might be useful, for example, when testing.

    Now is probably a reasonable time to observe that Go uses static typing, but := allows you to create typed variables as you go, so long as you use = the next time the variable value is set.

2. We use the `go` keyword when calling application.Monitor to create a goroutine, which is a lightweight function that can run concurrently with other functions. This is one of the key features and strengths of the Go language - it has built-in concurrency which allows Go programs to perform many tasks simultaneously without much overhead. This contrasts with Python, where external libraries such as asyncio are needed to provide an event loop for managing asynchronous tasks, which is more complex and involves more overhead.

    In this case, it lets us run the application.Monitor function while continuing to run the main() function. The main() function creates a channel called signalChan that monitors for the interupt signal that occurs when you press "Ctrl+C" in a terminal. So long as nothing is added to the channel, the function will simply wait at the <- point. This means the gameEvents channel which was created earlier will remain open indefinitely.
	
The orchestrate.go file contains the main loop which checks for windows opening and closing and applies the logic of whether to force close the lobby window. The file is actually somewhat large, so for the purpose of this blog I've removed some elements (constant values, default ProcessRetriever and ProcessKiller values, some comments and logging, live updating of config values if the file changes etc). If interested, you can see the full code using the repo linked at the top of this post.

The main function, Monitor, will use two helper functions. The first helper function encapsulates the logic of determining whether we're allowed to open the lobby or not:

```go
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

The second helper function applies the logic to decide whether a game counts (or is a remake - a brief game where a player fails to join and the game is not played out), update the game counter and determine the duration of the break based on whether it's just the end of a game or the end of session:

```go
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

We see some more time-based operations here: 
* `gameEndTime.Sub(gameStartTime)` calculates the game duration as gameEndTime - gameStartTime
* `gameEndTime.Add(breakDuration)` calculates the end-of-break time as gameEndTime + breakDuration
* `time.Duration(cfg.BreakBetweenSessionsMinutes)` converts an integer (e.g. 15) into a time value (15 nanoseconds), which is multiplied by the number of nanoseconds in a minute (`time.Minute`)

The two helper functions are called from the main Monitor function below. Note that this function starts with an uppercase letter, whereas the helper functions above started with lower case letters. This means unlike the other two functions, this function will appear and be available when importing this package within main.go:

```go
func Monitor(cfg config.Config, events chan window.ProcessEvent, pr window.ProcessRetriever, pk window.ProcessKiller) {

	go window.MonitorProcess(LOBBY_WINDOW_NAME, events, CHECK_FREQUENCY_SECONDS, pr)
	go window.MonitorProcess(GAME_WINDOW_NAME, events, CHECK_FREQUENCY_SECONDS, pr)

	var gameStartTime, gameEndTime, endOfBreak time.Time
	var breakDuration time.Duration
	var sessionGames int

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
We start by making two calls to `go window.MonitorProcess()` - one for the lobby window and one for the game window - which run concurrently as we're calling them as goroutines using the special `go` keyword. When I first got my prototype code working, I actually had a single function that monitored for both window names in the SELECT statement. When I refactored the window package to be more general purpose, I decided to keep it simple and only monitor for a single process. Hence, at this point we make two calls to the function. We could probably refactor the window code to be able to monitor for multiple processes at a time if we needed maximum efficiency, at the cost of additional complexity.

Next we initiate the `for event := range events {` loop, which operates on the events channel which is being kept open by the main.go program that calls this function. As long as the channel remains open, this loop will continue to run, processing events as they are added to the channel. This is a really interesting aspect of the Go language compared to Python. In the Python environment, concurrency requires importing a library (e.g. asyncio) that will run the loop to monitor for events and call a *callback function* when an event being monitored for occurs. These functions are *cooperatively shceduled*, which means they only yield control when explicitly told to (e.g. using `await` in asyncio), potentially leading to one function being unintentionally blocked by another. In the Go environment, concurrency is a core primative, so no additional library is required. Goroutine functions are *preemptively scheduled*, which means the runtime can stop one function to run another one at any point as needed.

Within the loop itself, we have a few core actions:

* We start by checking if there has been a significant gap between games, in which case we reset the session counter. This avoids the scenario where someone doesn't play all the games in their session, then comes back to a reduced allowance in the next session

* If the event is a game starting, we call the `window.WaitForProcessClose` function to check more frequently (every 1 second) for the game ending. This function is intentionally *not* called as a goroutine, so the regular monitoring (every 15 seconds) isn't called unnecessarily while the game is active.

* If the event is a game ending, we call the *postGame* helper function described earlier, to update the session game count and determine the break duration, then close the lobby window, potentially with a short delay.

* If the event is a lobby opening, we call the *isLobbyBan* helper function described earlier, and close the lobby if a break is being enforced.

This covers off the main application and domain logic. Next, we'll discuss the remaining packages that make up the complete application.

### Utils

This tiny package has a single function in it which is used by both the Config and Notifications packages, to return the directory and/or filename of the currently running executable.

```go
func GetCurrentPath(includeFilename bool) (string, error) {
	exePath, err := os.Executable()
	if err != nil {
		exePath, err = os.Getwd()
		if err != nil {
			return "", err
		}
	}

	exePath, err = filepath.Abs(exePath)
	if err != nil {
		return "", err
	}

	if includeFilename {
		return exePath, nil
	}
	return filepath.Dir(exePath), nil
}
```

While it seems somewhat trivial, this function was added to resolve an interesting practical implementation issue. When a compiled .exe file is run manually, the working directory is the same directory as the .exe file, but this is not the case when the .exe file is run as a result of a Windows registry key. For this reason, the config settings, which are located in the config.json file within the same folder as the .exe file, won't be picked up correctly if only the file name is used; a full path is required. In this function, we use `os.Executable()` to get the path of the running executable, with a fallback to the current working directory (e.g. if using `go run`), convert this to an absolute path, and optionally exclude the filename and just return the path using `.Dir`.

### Config

The config package acts both as a central source for the inputs required for the other packages, as well as the main way that users can interact with the application, as the application is designed to run silently in the background.

We start by defining a Config struct, which houses the key values that drive our application logic. We also create a defaultConfig variable, which is a Config with hard coded values. This is *somewhat* analogous to the Python concept of creating a class and initialising its parameters with an \_\_init\_\_ function.

```go
type Config struct {
	BreakBetweenGamesMinutes    int    `json:"breakBetweenGamesMinutes"`
	BreakBetweenSessionsMinutes int    `json:"breakBetweenSessionsMinutes"`
	GamesPerSession             int    `json:"gamesPerSession"`
	MinimumGameDurationMinutes  int    `json:"minimumGameDurationMinutes"`
	LobbyCloseDelaySeconds      int    `json:"lobbyCloseDelaySeconds"`
	DailyStartTime              string `json:"dailyStartTime"`
	DailyEndTime                string `json:"dailyEndTime"`
	LoadOnStartup               bool   `json:"loadOnStartup"`
}

// DailyStartTime and DailyEndTime e.g. "04:00" "22:00"
var defaultConfig = Config{
	BreakBetweenGamesMinutes:    5,
	BreakBetweenSessionsMinutes: 60,
	GamesPerSession:             3,
	MinimumGameDurationMinutes:  15,
	LobbyCloseDelaySeconds:      30,
	DailyStartTime:              "00:00",
	DailyEndTime:                "00:00",
	LoadOnStartup:               true,
}
```

Next we define a function to save config json files:

```go
func defaultPath(filename string) (string, error) {
	if filename == "" {
		exePath, err := utils.GetCurrentPath(false)
		if err != nil {
			return "", err
		}
		filename = filepath.Join(exePath, "config.json")
	}
	return filename, nil
}

func SaveConfig(filename string, cfg Config) error {

	// use default path and filename unless specified explicitly
	filename, err := defaultPath(filename)
	if err != nil {
		return err
	}

	// create config file
	file, err := os.Create(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	// save cfg in json format
	encoder := json.NewEncoder(file)
	encoder.SetIndent("", "  ") // Pretty-print JSON
	return encoder.Encode(cfg)
}
```

The defaultPath function uses the GetCurrentPath function from the utils package to create a full path to the config.json file in the directory where the code is being run.

The SaveConfig function uses `os.Create` to create a new file. Assuming this works, the `defer file.Close()` line will close the file at the end of the function. This is a nice feature of Go, as you don't require an additional line to close the file, or an extra indenting level as would be the case using `With` when opening a file in Python. It also ties in with Go's *panic* failure mode, which is used to gracefully unwind the stack when there's an unexpected failure, so the file will still be closed when an unexpected error occurs. The function then uses a json encoder to encode the Config value into a json formatted file.

Next we have a function to load config files:

```go
func LoadConfig(filename string) (Config, error) {

	// use default path and filename unless specified explicitly
	filename, err := defaultPath(filename)
	if err != nil {
		return Config{}, err
	}

	// open file, creating a new config file if one is not found
	file, err := os.Open(filename)
	if err != nil {
		log.Println("Config file not found, creating default config...")
		SaveConfig(filename, defaultConfig)
		return defaultConfig, nil
	}
	defer file.Close()

	// load json into config
	decoder := json.NewDecoder(file)
	config := defaultConfig
	err = decoder.Decode(&config)
	if err != nil {
		log.Printf("Error when decoding config file: %v", err)
		log.Println("Invalid config file, resetting to default values...")
		SaveConfig(filename, defaultConfig)
		return defaultConfig, nil
	}

	return config, nil
}
```

This function attempts to open the config file with `os.Open`. If it doesn't exist - which would typically only be the case when a compiled .exe is downloaded as a single file - it will use the defaultConfig variable and the SaveConfig function to create a new one. The function then uses a decoder to read the json file into a Config struct.

The config package also includes a small `CheckConfigUpdated` function, which is used to ensure changes made to the config file are picked up while the program is running:

```go
func CheckConfigUpdated(filename string, t time.Time) (bool, error) {

	// use default path and filename unless specified explicitly
	filename, err := defaultPath(filename)
	if err != nil {
		return false, err
	}

	// check if date modified is after time t
	fileInfo, err := os.Stat(filename)
	if err != nil {
		if os.IsNotExist(err) {
			return false, fmt.Errorf("unable to find config file: %s", filename)
		}
		return false, err
	}

	// Compare the file's modification time with the provided time
	return fileInfo.ModTime().After(t), nil
}
```

This function uses `os.Stat` to check the last modified time, and if this fails, uses `os.IsNotExist` to check if the file exists and provide a more useful error message. This was useful, for example, when debugging the temporary folder issue described in the [utils](#utils) section.

### Startup

The startup package adds or removes an item in the Windows registry, which allows the program to start automatically when the user logs into Windows.

Using the Windows registry proved to be surprisingly easy. I initially tried to use Windows Task Scheduler for this purpose (using `exec.Command("schtasks", ...)`), but this approach proved ineffective; the Microsoft documentation was lacking and even when creating user-only tasks with /RU, it seemed to require admin privileges.

The core code is in the *addToStartup*, *removeFromStartup* and *startupEntryExists* helper functions:

```go
import (
	"golang.org/x/sys/windows/registry"
)

// create registry key
func addToStartup(taskName, exePath string) error {
	key, _, err := registry.CreateKey(
		registry.CURRENT_USER,
		`Software\Microsoft\Windows\CurrentVersion\Run`,
		registry.SET_VALUE,
	)
	if err != nil {
		return fmt.Errorf("failed to open registry key: %v", err)
	}
	defer key.Close()

	err = key.SetStringValue(taskName, exePath)
	if err != nil {
		return fmt.Errorf("failed to set registry value: %v", err)
	}

	slog.Info("Successfully added to startup", "taskName", taskName, "exePath", exePath)
	return nil
}

// remove registry key
func removeFromStartup(taskName string) error {
	key, err := registry.OpenKey(
		registry.CURRENT_USER,
		`Software\Microsoft\Windows\CurrentVersion\Run`,
		registry.SET_VALUE,
	)
	if err != nil {
		return err
	}
	defer key.Close()

	return key.DeleteValue(taskName)
}

// check if regkey exists, returns a bool and the exePath if it exists
func startupEntryExists(taskName string) (bool, string, error) {
	key, err := registry.OpenKey(
		registry.CURRENT_USER,
		`Software\Microsoft\Windows\CurrentVersion\Run`,
		registry.QUERY_VALUE,
	)
	if err != nil {
		return false, "", err
	}
	defer key.Close()

	exePath, _, err := key.GetStringValue(taskName)
	if err == registry.ErrNotExist { // check worked and regkey doesn't exist
		return false, "", nil
	}
	if err != nil { // check failed
		return false, "", err
	}
	return true, exePath, nil // check worked and regkey exists
}
```

These functions use `registry.OpenKey` to access an entry at `Software\Microsoft\Windows\CurrentVersion\Run` in the Windows registry, which lets us add, edit or remove a key, which drives the automatic starting of our program when the user logs in.

Two functions, `ConfirmLoadOnStartup` and `ConfirmNoLoadOnStartup` use these functions to ensure the registry key either exists and is up to date, or does not exist, based on the preference in the config file. The logic is fairly simple, so I've omitted these from the write up. If interested, you can see the full code using the repo linked at the top of this post.

### Notifications

The core code used to send Windows toast notifications is relatively straightforward:

```go
import (
	"github.com/go-toast/toast"
)

func SendNotification(title, message string, duration_long bool) {
	// short duration is ~7 sec, long duration is ~25 sec
	duration := toast.Short
	if duration_long {
		duration = toast.Long
	}
	notification := toast.Notification{
		AppID:   "LoL Monitor",
		Title:   title,
		Message: message,
		Duration: duration,
	}
	err := notification.Push()
	if err != nil {
		log.Printf("Failed to send notification: %v", err)
	}
}
```

While SendNotification can be used directly by other packages, it's actually used mainly by the notifications package itself, which includes additional functions to send notifications when the lobby is about to close, is being closed at the end of a game, or is being closed due to a ban being enforced. Having these functions within the notifications package makes it easier to find, test and adjust them. These functions are then called by the orchestrate package as needed. For example, here is one such function:

```go
func LobbyBlocked(endOfBreak time.Time) {
	notification_title := "You're on a break!"
	notification_text := fmt.Sprintf("Try again in %.0f minutes at %v",
		math.Ceil(time.Until(endOfBreak).Minutes()),
		endOfBreak.Format("3:04pm"))
	SendNotification(notification_title, notification_text, false)
}
```

That's the last of the Go code. Next we'll briefly touch on some additional things that were needed to finish off the project.

## Moving from prototype to polished product

While it's great to have a working application, it's even better to make it accessable to others. This mainly involved updating the readme file within the Github repo to include useful information, including what the default settings in the config.json file do and how to customise them, as well as answers to questions that users of the application might have. This information can be found in the readme of the repo linked at the top of this post, and is replicated below.

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

Given my personal interest in the subject matter, I decided it was only appropriate I did a bit of [dogfooding](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) before I finished this blog post. This served to iron out a few wrinkles in the product - such as the config file [not being picked up]((#utils)) when the program was run by the Windows registry - and improve its design - primarily the layout and contents of the toast notifications.

My first-hand experience of the application produced some interesting results:

1. Despite going into the experiment with a sense of fear and skepticism - anticipating that I would just override the tool - I actually found it to be surprisingly effective at helping me to resist the urge to play more than I wanted to. There was a real opportunity at the end of each game to reassess whether I wanted to continue playing and consider alternatives. With my cognitive resources running low at the end of a game, the easy and mindless option - which was previously clicking the "Continue" button - was now to let the program close itself, and take a short break.

2. I felt a surprising sense of *relief* when coming out of emotionally charged games, knowing that I was going to take a small break. I was more conscious of how I was feeling physically and emotionally, rather than being caught up in the outcome of the game. I felt much more calm and focused going into the next game.

3. Just as it's easy to log in or click the "Continue" button out of habit, it quickly became a habit to take a break between games. Even when I manually closed lolmonitor.exe while I was debugging and despite it not being enforced, I still felt like taking a short break.

It was quite useful to go through the process of being a user and a developer at the same time. More generally, the closer you can get to your customers, the better the product you can create. Being a customer yourself is about as close as you can get.

## Conclusion

I had a lot of fun building this application. It was my first experience of the Go programming language, and I made the most of it, delving into project structuring best practices and leveraging interesting packages that let me use the Windows registry and toast notification functionality.

The next step for this project is to reach out to the folks at [Broken by Concept](https://weteachleague.com/podcast/) and see if any of their members are interested in trying it out. It would be good to get some beta testers to ensure the application is as bullet proof as possible. It's also currently only available to Windows users, but there's potential to develop a Mac version.

That's all for now. I hope you've enjoyed the insights from this project as much as I enjoyed building it!

---
<small>Footnotes:</small>

[^1]: Interestingly, the term Mock seems to be used loosely in the Go community. Technically a Mock is a configurable double that fails the test if its expectations aren't met. Often what is being referred to is actually a Fake (e.g. in-memory database instead of a real database), a Stub (returns pre-defined data) or a Spy (records calls). In my case, the window interface is actually a Fake.