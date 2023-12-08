+++
title = 'The Actor Model in a C# World'
date = 2023-12-07T17:46:43-05:00
draft = false
+++

*This post is the first in an occasional series on building a distributed system using Akka.NET*

The Actor Model is one of those approaches to system design that, at first, struck me as overly complex, but as I learned more about it, I discovered its simplicity and elegance, and its power. The [Actor Model](https://en.wikipedia.org/wiki/History_of_the_Actor_model) goes back to the 1970s with [Carl Hewitt](https://en.wikipedia.org/wiki/Carl_Hewitt) and has gained popularity over time for its ability to provide a simple approach to building highly scalable and concurrent systems.

(If you want an enjoyable video showing some greats having a discussion about the Actor Model, check out this [informal whiteboard chat](https://youtu.be/1zVdhDx7Tbs?si=bK2tx_0XIKC-10ak) with Erik Meijer, Carl Hewitt, and Clemens Szyperski.)

## What is the Actor Model
So briefly, what is the Actor Model and why should I use it? If you've had to design a system where you need to create a number of threads to handle some type of concurrent processing and ensure that you don't paint yourself into a deadlock corner, the Actor Model could be a tool to help solve that problem.

Actor Model systems are characterized by communication via immutable messages between objects called "actors". An actor is generally a small, lightweight object that should be specialized in its scope and can have its own state. An actor can communicate with other actors via messages, which are immutable POCOs. Actors are lightweight, cheap to spin up, and should be easy to tear down. Actor systems can see hundreds, thousands or more actors running at any given time.

Actors, in some actor model frameworks, can create actors of their own ("children") and can supervise those child actors. These supervisor actors can determine to tear down, restart, or ignore child actors that potentially get into a bad state.

## Hello Actor!

So let's see the codez, right!! Let's use Akka.NET 1.5 to create a simple Actor System:

``` csharp
var system = ActorSystem.Create("helloAkka");
```

And let's create the top level supervisor and pass a file to it that we want to get a word count:

```csharp
var counter = system.ActorOf(CountSupervisor.Create(), "supervisor");
counter.Tell(new StartCount(file));
```

What does that message look like? Well it's a basic POCO class with a simple property on it:

```csharp
public class StartCount(String file)
{
    public readonly String FileName = file;
}
```

And the `CountSupervisor` actor? There's a bit here, so let's take a look at what's going on:

```csharp
public class CountSupervisor : ReceiveActor
{
    public static Props Create()
    {
        return Props.Create(() => new CountSupervisor());
    }

    private Dictionary<String, Int32> _wordCount;
    private readonly Int32 _numberOfRoutees;
    private Int32 _completeRoutees;

    public CountSupervisor()
    {
        _wordCount = new Dictionary<String, Int32>();
        _numberOfRoutees = 5;
        _completeRoutees = 0;

        SetupBehaviors();
    }

    private void SetupBehaviors()
    {
        Receive<StartCount>(msg =>
        {
            var fileInfo = new FileInfo(msg.FileName);
            var lineNumber = 0;

            var lineReader = Context.ActorOf(new RoundRobinPool(_numberOfRoutees)
                                .Props(LineReaderActor.Create()));

            using (var reader = fileInfo.OpenText())
            {
                while (!reader.EndOfStream)
                {
                    lineNumber++;

                    var line = reader.ReadLine() ?? String.Empty;
                    lineReader.Tell(new ReadLineForCounting(lineNumber, line));
                }
            }

            lineReader.Tell(new Broadcast(new Complete()));
        });

        Receive<MappedList>(msg =>
        {
            foreach (var key in msg.LineWordCount.Keys)
            {
                if (_wordCount.ContainsKey(key))
                {
                    _wordCount[key] += msg.LineWordCount[key];
                }
                else
                {
                    _wordCount.Add(key, msg.LineWordCount[key]);
                }
            }
        });

        Receive<Complete>(msg =>
        {
            _completeRoutees++;

            if (_completeRoutees == _numberOfRoutees)
            {
                var topWords = _wordCount.OrderByDescending(w => w.Value).Take(25);
                foreach (var word in topWords)
                {
                    Console.WriteLine($"{word.Key} == {word.Value} times");
                }
            }
        });
    }
}
```

And that mysterious LineReaderActor? Here we go:

```csharp
public class LineReaderActor  : ReceiveActor
{
    public static Props Create()
    {
        return Props.Create(() => new LineReaderActor());
    }

    public LineReaderActor()
    {
        SetupBehaviors();
    }

    private void SetupBehaviors()
    {
        Receive<ReadLineForCounting>(msg =>
        {
            var cleanFileContents = Regex.Replace(msg.Line, @"[^\u0000-\u007F]", " ");

            var wordCounts = new Dictionary<String, Int32>();

            var wordArray = cleanFileContents.Split(new char[] { ' ' }, 
                StringSplitOptions.RemoveEmptyEntries);
            foreach (var word in wordArray)
            {
                if (wordCounts.ContainsKey(word))
                {
                    wordCounts[word] += 1;
                }
                else
                {
                    wordCounts.Add(word, 1);
                }
            }

            Sender.Tell(new MappedList(msg.LineNumber, wordCounts));
        });

        Receive<Complete>(msg =>
        {
            Sender.Tell(msg);
        });
    }
}
```

## Taking Care of Your Children
