> [!NOTE]
> This fork is exclusively designed to be used within **framerate-dependent** games. It does not support or account for variable framerate in any way.
> If you prefer to use something that accounts for delta time, check out one of the parent repositories that this fork is based on.
>
> Additionally, this fork may not work at all in .net 7.0 or older due to the usage of the newer .net 8.0 syntax.

# Coroutines
A simple system for running nested coroutines in C#. Just drop `Coroutines.cs` into your project and you're ready to go.

## What is a coroutine?
C# has a feature called "enumerators" which are functions that can be *suspended* during their execution. They do this by using `yield return`, rather than regular `return`. When you use `yield`, you effectively *pause* the function, and at any time you can *resume* it and it will continue after the most recent `yield`.

In this context, a "coroutine" is basically one of these functions, but wrapped up in a nice little package and a few extra features:

- A container that will automatically update several coroutines for you
- A simple syntax for nesting coroutines so they can yield to others
- The ability to yield an integer number, which will pause the routine for a desired amount of updates
- Handles and names to routines are provided to make tracking them easier

## Example usage
To use the system, you need to create a `CoroutineRunner` and update it at regular intervals (eg. in a game loop).

```csharp
CoroutineRunner runner = new CoroutineRunner();

void UpdateGame()
{
    runner.Update();
}
```

Now you can run coroutines by calling `Run()` or `TryRun()`. Here's a simple coroutine that counts:

```csharp
IEnumerator CountTo(int num, int delay)
{
    for (int i = 1; i <= num; ++i)
    {
        yield return delay;
        Console.WriteLine(i);
    }
}
void StartGame()
{
    // Count to 10, pausing 60 frames between each number
    runner.Run("counting", CountTo(10, 60), 0);
}
```

When you yield an integer number, it will pause the coroutine for that many frames.

You can also nest coroutines by yielding to them. Here we will have a parent routine that will run several sub-routines:

```csharp
IEnumerator DoSomeCounting()
{
    Console.WriteLine("Counting to 3 slowly...");
    yield return CountTo(3, 120);
    Console.WriteLine("Done!");

    Console.WriteLine("Counting to 5 normally...");
    yield return CountTo(5, 60);
    Console.WriteLine("Done!");

    Console.WriteLine("Counting to 99 quickly...");
    yield return CountTo(99, 6);
    Console.WriteLine("Done!");
}

void StartGame()
{
    runner.Run("my_coroutine", DoSomeCounting(), 0);
}
```

You can also stop any running routines:

```csharp
// Stop all running routines
runner.StopAll();

// Start a routine and store a handle to it
CoroutineHandle myRoutine = runner.Run("some_routine", SomeRoutine(), 0);

// Alternately:
bool success = runner.TryRun("some_routine", SomeRoutine(), 0, out CoroutineHandle myRoutine);

// Stop a specific routine
myRoutine.Stop();

// Alternatively:
runner.Stop("some_routine");
```

## Other tips and tricks
A coroutine can run infinitely as well by using a loop. You can also tell the routine to "wait for the next update" by yielding `null`:

```csharp
IEnumerator RunThisForever()
{
    while (true)
    {
        yield return null;
    }
}
```

Coroutines are very handy for games, especially for sequenced behavior and animations, acting sort of like *behavior trees*. For example, a simple enemy's AI routine might look like this:

```csharp
IEnumerator EnemyBehavior()
{
    while (enemyIsAlive)
    {
        yield return PatrolForPlayer();
        yield return Speak("I found you!");
        yield return ChaseAfterPlayer();
        yield return Speak("Wait... where did you go!?");
        yield return ReturnToPatrol();
    }
}
```

Sometimes you might want to run multiple routines in parallel, and have a parent routine wait for them both to finish. For this you can use the return handle from `Run()` or `TryRun()`:

```csharp
IEnumerator GatherNPCs(Vector gatheringPoint)
{
    //Make three NPCs walk to the gathering point at the same time
    var move1 = runner.Run("npc1_gather", npc1.WalkTo(gatheringPoint), 0);
    var move2 = runner.Run("npc2_gather", npc2.WalkTo(gatheringPoint), 0);
    var move3 = runner.Run("npc3_gather", npc3.WalkTo(gatheringPoint), 0);

    //We don't know how long they'll take, so just wait until all three have finished
    while (move1.IsRunning || move2.IsRunning || move3.IsRunning)
        yield return null;

    //Now they've all gathered!
}
```

Here is a more complicated example where I show how you can use coroutines in conjunction with asynchronous functions (in this case, to download a batch of files and wait until they've finished):

```csharp
IEnumerator DownloadFile(string url, string toFile)
{
    using var client = new HttpClient();
    var downloadTask = client.GetStreamAsync(package.DownloadUrl);
    while (!downloadTask.IsCompleted)
        yield return null;

    if(!download.IsCompletedSuccessfully)
    {
        Console.WriteLine($"Failed to download {url}!");
        yield break;
    }

    using var file = File.Open(toFile, FileMode.Create);
    var copyTask = task.Result.CopyToAsync(file);
    while (!copyTask.IsCompleted)
        yield return null;
}

//Download the files one-by-one in sync
IEnumerator DownloadOneAtATime()
{
    yield return DownloadFile("https://site.com/file1.png", "file1.png");
    yield return DownloadFile("https://site.com/file2.png", "file2.png");
    yield return DownloadFile("https://site.com/file3.png", "file3.png");
    yield return DownloadFile("https://site.com/file4.png", "file4.png");
    yield return DownloadFile("https://site.com/file5.png", "file5.png");
}

//Download the files all at once asynchronously
IEnumerator DownloadAllAtOnce()
{
    //Start multiple async downloads and store their handles
    var downloads = new List<CoroutineHandle>();
    downloads.Add(runner.Run("download_file1", DownloadFile("http://site.com/file1.png", "file1.png"), 0));
    downloads.Add(runner.Run("download_file2", DownloadFile("http://site.com/file2.png", "file2.png"), 0));
    downloads.Add(runner.Run("download_file3", DownloadFile("http://site.com/file3.png", "file3.png"), 0));
    downloads.Add(runner.Run("download_file4", DownloadFile("http://site.com/file4.png", "file4.png"), 0));
    downloads.Add(runner.Run("download_file5", DownloadFile("http://site.com/file5.png", "file5.png"), 0));

    //Wait until all downloads are done
    while (downloads.Count > 0)
    {
        yield return null;
        for (int i = 0; i < downloads.Count; ++i)
            if (!downloads[i].IsRunning)
                downloads.RemoveAt(i--);
    }
}
```

## Why coroutines?

I use coroutines a lot in my games, as I find them great for organizing actor behavior and animations. As opposed to an async callback-based system, coroutines allow you to write your behaviors line-by-line, like how you would naturally write code, and result in very clean and easy to understand sequences.

There are good and bad times to use them, and you will get better at distinguishing this as you use them more. For many of my games, coroutines have been completely priceless, and have helped me organize and maintain very large and complicated systems that behave exactly in the order I wish them to.

**NOTE:** Not all languages have built-in support for coroutine systems like this. If you plan on porting your code to other languages, it may not be worth the pain of porting if your target language does not have a reliable means of implementing coroutines.
