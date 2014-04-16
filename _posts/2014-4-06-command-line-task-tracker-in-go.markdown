---

layout: post
title: Writing a Command-line Task Tracker in Go

---
Interested in the Go programming language? This tutorial will show you how to write a simple command-line task tracker using Go. It is assumed you have completed the [tour](http://tour.golang.org/#1) and have a [working Go environment already installed](http://golang.org/doc/install).

Our program will work similarly to [todo.txt](http://todotxt.com). 

When done, we will be able to add, list, and complete tasks from the command line using the command "todo":

    $ todo ls
    [1]     [2014-3-27]     Get groceries
    [2]     [2014-3-27]     Fix Issue #4501
    [3]     [2014-3-28]     Add more features to
    $ todo add "Update readme file"
    Task is added: Update readme file
    $ todo ls
    [1]     [2014-3-27]     Get groceries
    [2]     [2014-3-27]     Fix Issue #4501
    [3]     [2014-3-28]     Add more features to
    [4]     [2014-3-29]     Update readme file
    $ todo complete 1
    Task Marked as complete: Get groceries
    $ todo ls
    [1]     [2014-3-27]     Fix Issue #4501
    [2]     [2014-3-28]     Add more features to
    [3]     [2014-3-29]     Update readme file

By implementing these functions, you will learn how to persist user inputted data on the file system, display that data in a useful form, and modify that data based on certain rules.

## Table of Contents
	1. ####Gettine Started With CLI
	2. ####Adding Tasks, Persistence With JSON
	3. ####Listing Tasks
	4. ####Completing Tasks

## Getting Started With CLI

First get the [cli package by Codegangsta](https://github.com/codegangsta/cli), this will make it simple for us to wrap the application's functionality around a command line interface.

    go get github.com/codegangsta/cli

If you visited the above link and read the README, you'll notice there's already code there that we can reproduce to get the basic commands of our app working. 

Create a file called todo.go:
{% highlight go %}
package main

import (
    "fmt"
    "github.com/codegangsta/cli"
    "os"
)

func main() {
    app := cli.NewApp()
    app.Name = "todo"
    app.Usage = "add, list, and complete tasks"
    app.Commands = []cli.Command{
	    {   
		    Name:      "add",
		    Usage:     "add a task",
		    Action: func(c *cli.Context) {
			    fmt.Println("added task: ", c.Args().First())
		    },  
	    },  
	    {   
		    Name:      "complete",
		    Usage:     "complete a task",
		    Action: func(c *cli.Context) {
			    fmt.Println("completed task: ", c.Args().First())
		    },  
	    },  
    }   
    app.Run(os.Args)
}
{% endhighlight %}

`cli.NewApp()` returns a pointer to an App struct. The App struct acts as a wrapper for our program's functionality and metadata. There are a number of attributes we can modify as you can see in the source code for package cli [here](https://github.com/codegangsta/cli/blob/f7c1cd9a11e75b5ad458628188f733a325e14ca5/app.go#L10-L34), however we will be using just `Name`, `Usage`, and `Commands` for now.

`app.Commands = []cli.Command {....}` assigns to our app struct an array of type `Command` (original struct definition is [here](https://github.com/codegangsta/cli/blob/f7c1cd9a11e75b5ad458628188f733a325e14ca5/command.go#L9-L23)). A Command is also a struct, the `Name` in this case defines what subcommand will run the anonymous function defined in `Action`, so running:
    
    $ godo run todo.go add "Hello World!"
    
will print:

    added task: Hello World!
    
Obviously this is not very useful, as the task isn't actually stored anywhere. Let's consider now how we can persist tasks.

## Adding Tasks, Persistence with JSON

Go ships with an excellent [JSON](http://www.json.org/) library that we will be leveraging to store our task list as a file.

The json package provides us the ability to convert structs into json text data. So if we define a Task struct:

{% highlight go %}

type Task struct {
    Content  string
    Complete bool
}

{% endhighlight %}

We can use the `Marhsal` function to convert a Task into json:

{% highlight go %}
m := Task{Content: "Hello", Complete: true}
b, error := json.Marshal(m)

{% endhighlight %}
`b` is now a byte slice that holds the JSON text `{"Content":"Hello","Complete":true}`, simple as that!

To get started, add our Task struct under the import lines of todo.go so it looks like this:

{% highlight go %}
import (
    "fmt"
    "github.com/codegangsta/cli"
    "os"
)

type Task struct {
    Content  string
    Complete bool
}
....

{% endhighlight %}

Now we need to build an instance of Task based off of the user's input. To do this we will modify the Action in our "add" command.
    
{% highlight go %}
...
    app.Commands = []cli.Command{
	{
		Name:      "add",
		ShortName: "a",
		Usage:     "add a task to the list",
		Action: func(c *cli.Context) {
			task := Task{Content: c.Args().First(), Complete: false}
			fmt.Println(task)
		},
	},
...

{% endhighlight %}

Now if we run `go run todo.go add "hello!"` we will be given the output: `hello! false`(fmt.Println is printing our struct without field names, you can see field names by using `fmt.Printf("%+v", task)` instead)

Now to persist it as a json file, add the following lines(and make sure to add `io/ioutil` to your imports):

{% highlight go %}
task := Task{Content: c.Args().First(), Complete: false}
j, err := json.Marshal(task)
if err != nil {
    panic(err)
}
ioutil.Write("tasks.json", j, 0600)
{% endhighlight %}

Now if you add a task it will write the task to the json file in  the same directory as your program.

However, as you may have noticed, `ioutil.Write` truncates(deletes) the existing tasks.json file to write the new byte slice. Now technically we could read the contents of the tasks.json, load it into a variable(store the whole file in memory), combine the old contents with the newly added json, and write that to a file. This works fine for a small number of tasks, but what if we had 10 million? We cannot say for sure that a user won't be loading several terabytes of tasks(which would take up a lot of space in memory). So to ensure our code is sane, and is ready for any enterprise/government level task-tracking, we'll add some more lines for appending to the file.

In order to append, we will use `os.OpenFile` passing the `os.O_APPEND` option. As `os.OpenFile` returns an error if the file doesn't already exist, we have to supply the `os.O_CREATE` option as well to create the file if it doesn't exist:

{% highlight go %}
    Action: func(c *cli.Context) {
            task := Task{Content: c.Args().First(), Complete: false}
            j, err := json.Marshal(task)
            if err != nil {
                    panic(err)
            }
            // Add a newline to the json to be more human-readable
            j = append(j, "\n"...)
            // Open tasks.json with these options: append, write only, and create-if-not-exists
            f, _ := os.OpenFile("tasks.json", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0600)
            // Append our json to tasks.json
            if _, err = f.Write(j); err != nil {
                    panic(err)
            }
    },

{% endhighlight %}

Now whenever we run `todo add "task"`, our program will append the task to the end of the file.

In the interest of keeping our code organized, let's move this all into it's own function.

{% highlight go %}
    func AddTask(task Task) {
		j, err := json.Marshal(task)
		if err != nil {
			panic(err)
		}
		// Add a newline to the json to be more human-readable
		j = append(j, "\n"...)
		// Open tasks.json in append-mode.
		f, _ := os.OpenFile("tasks.json", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0600)
		// Append our json to tasks.json
		if _, err = f.Write(j); err != nil {
			panic(err)
		}
	}

{% endhighlight %}
We can now call `AddTask(task)`in our Action to append to the tasks file.

#Listing Tasks
---
Now that we can add tasks to our list, it would be somewhat useful if we could also view that without opening tasks.json manually.


We will add a new command to our code called "list".

{% highlight go %}
    {
            Name:       "list"
            ShortName:  "ls",
            Usage:      "print all uncompleted tasks in list",
            Action: func(c *cli.Context) {
	    	    ListTasks()
            }
    }

{% endhighlight %}

By specifying `ShortName`, we allow our users to to type "ls" instead of "list".

To print the tasks we will need to iterate over the tasks existing in tasks.json. 

Now, the most straight-forward approach would be to load the whole file into memory as a slice of tasks. But as mentioned earlier, we could be dealing with a very large file. Therefor it's preferable to load only one line of the file at a time, convert that into a task, and print that task.

To access the contents of the file line-by-line, we can use package bufio(buffered io). This will let us load each line into a buffer, without loading the entire file in memory. We will be using a bufio Scanner, which splits a file's contents into text tokens according to a delimiter(by default "\n").

{% highlight go %}
func ListTasks() {
	// Check to see if file exists or not
	if _, err := os.Stat("tasks.json"); os.IsNotExist(err) {
		log.Fatal("tasks file does not exist")
		return
	}
	file, err := os.Open("tasks.json")
	if err != nil {
		panic(err)
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	// Our index to keep track of the current task number
	i := 1
	// scanner.Scan() advances the scanner to the next token and returns true.
	// By default a token is a new-line delimited string.
	// When the scan stops(hits End Of File), it returns false.
	for scanner.Scan() {
		// scanner.Text() returns the current token in scanner as a string
		j := scanner.Text()
		t := Task{}
		// We pass to Unmarshall the json converted to a byte slice, as well as
		// a pointer to our empty task. It will then populate our task's fields
		// with our json values.
		err := json.Unmarshal([]byte(j), &t)
		// By default we'll only print tasks that are not complete
		if err != nil {
			panic(err)
		}
		if !t.Complete {
			fmt.Printf("[%d] %s\n", i, t.Content)
			i++
		}
	}
}

{% endhighlight %}
As explained in the comments, each time we call scanner.Scan() we are advancing the scanner to the next token. As a for loop with one condition runs until its expression evaluates to false, and Scan returns false when the scan is complete, our loop will continue executing until we hit the end of the file.

Now we can use `todo ls` to see all of our uncompleted tasks:

	$ todo ls
	[1] Task 1
	[2] Task two
	[3] Task number 3

# Completing Tasks

Lastly, we will be implementing functionality to mark a task as complete. Completion will be initiated by the following command

	todo complete #

\#  being the number of the task we want to complete. Note that this is the number as it appears when you run `todo ls`, not the actual index of the task as it appears in the tasks.json file. This is because when we print the tasks, we ignore completed ones, therefor our index is not incremented.

Now, there are a number of ways we can update the tasks file to reflect completed tasks. 

One way would be to load all of the tasks in the file into a Tasks slice, iterate over that slice checking for an uncompleted task with the right index, mark it as complete, delete the tasks file, and then write the new task to there. This isn't very clean and can become a problem with large files. 

What if then we treat the tasks file as text and search for the nth "bool" field that is true, and modify it to false?  Writing a good lexer or using regular expressions on this will be tricky. For example, what if a user does `todo add ""bool":true"`? You can never be too sure about these things. If we don't deal with the length of the old vs. new string when writing we can also corrupt the file. This approach is clearly a pain.

A safer way to handle this is to read each line of the task file and write them to a temp file, marking our completed task when we find it. Then we just need to swap the old file with the temp file.
The procedure will look like this:

1. Read each line of the tasks file.
2. Unmarshal the current line into a Task.
3. If the task is not complete, increment the index by 1.
	1. Check the index against the number given to us by the user. 
	2. If they match set Complete to true for the current task.
5. Marshall the task and write it to the temp file.
6. Once we've reached the end of the file, replace the original file with the temp file.

As you can see, a lot of the above functionality has already been implemented in our code. We will have to modify parts of it to reuse them.

## Writing to the file

For writing the completed tasks to the temp file we can leverage the AddTask function we wrote earlier. However, we will have to modify it to accept a new parameter specifying what file we want to write to(tasks.json or .temp).

     func AddTask(task Task, filename string) {
Then, inside AddTask() modify the line where we open the file from this:

{% highlight go %}
     f, _ := os.OpenFile("tasks.json", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0600)

{% endhighlight %}
to this:
      
{% highlight go %}
      f, _ := os.OpenFile(filename, os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0600)

{% endhighlight %}

Now, in our Command's Action, modify the call to include the filename:

{% highlight go %}
      AddTask(task, "tasks.json")
{% endhighlight %}

## Opening the file
As we need to open tasks.json in both ListTasks() and CompleteTasks(), we can move the code for that from ListTasks() into its own function as well:

{% highlight go %}
    func OpenTaskFile() *os.File {
        // Check to see if file exists or not
        if _, err := os.Stat("tasks.json"); os.IsNotExist(err) {
                log.Fatal("tasks file does not exist")
                return nil 
        }   
        file, err := os.Open("tasks.json")
        if err != nil {
                panic(err)
        }   
        return file
    }
{% endhighlight %}

This is what ListTasks() looks like after modifying it to use `OpenTaskFile()`: 

{% highlight go %}
    func ListTasks() {
        file := OpenTaskFile()
        defer file.Close()
        scanner := bufio.NewScanner(file)
        i := 1
        for scanner.Scan() {
                j := scanner.Text()
                t := Task{}
                err := json.Unmarshal([]byte(j), &t)
                if err != nil {
                        panic(err)
                }
                if !t.Complete {
                        fmt.Printf("[%d] %s\n", i, t.Content)
                        i++
                }
        }
   }
{% endhighlight %}

Much cleaner.

CompleteTask() will take an integer called idx(for index) from the user. As the flag from `c.Args().Flag()` will be a string, we have to convert it into an int. To do so we will import the strconv package:

{% highlight go %}
    import (
    ...
        "strconv"
    ...
    )
{% endhighlight %}

Next we'll use the `strconv.Atoi()` function to convert our string to type int and supply that to `CompleteTask()`, like so:

{% highlight go %}
    {
            Name:  "complete",
            Usage: "complete a task",
            Action: func(c *cli.Context) {
                    idx, err := strconv.Atoi(c.Args().First())
                    if err != nil {
                            panic(err)
                    }
                    CompleteTask(idx)
            },
    },
{% endhighlight %}

Now we can write the code for CompleteTask:

{% highlight go %}
func CompleteTask(idx int) {
    file := OpenTaskFile()
    defer file.Close()
    scanner := bufio.NewScanner(file)
    i := 1
    for scanner.Scan() {
	    j := scanner.Text()
	    t := Task{}
	    err := json.Unmarshal([]byte(j), &t) 
	    if err != nil {
		    panic(err)
	    }
	    if !t.Complete {
		    if idx == i {
			    t.Complete = true
		    }
		    i++
	    }
	    // Append the current task to the tempfile.
	    // Note the first time we call this it will actually 
	    // create the file AND write the task.
	    AddTask(t, ".tempfile")
    }
    // Now that tempfile is complete, overwrite its contents onto tasks.json
    os.Rename(".tempfile", "tasks.json")
    // We can now delete .tempfile
    os.Remove(".tempfile")
}
{% endhighlight %}

Our work is now complete, we can now add, list, and complete tasks using our command-line task tracker!

As you can see, the loop for reading each line in CompleteTask() is identical to that in ListTasks() up until `if !t.Complete {`. In another post I will cover how to move this into its own function and use closures to even further reduce code duplication. Also, you may have noticed that unlike the original demo at the top, we don't have the month/date the task was added to the list yet either! This will be covered in the next post as well(which I promise will be much shorter!)

See you again!
