---
layout: post
title:  "Addressing Memory Issues In Denaro"
date: '2023-01-16 09:40:00 -0500'
---

🪙 **Let's talk about memory mangement in Denaro,** and our fix for [#208](https://github.com/nlogozzo/NickvisionMoney/issues/208) 🛠️.

## Managed vs Unmanaged Languages 👾

Denaro was recently rewritten in C#, a managed language running on the dotnet runtime. "What's a managed language?" you may ask. Great question! A managed language differs than that of C, C++, and Rust (known as unmanaged) in that it has a garbage collector. A garbage collector automatically frees the memory of objects on the heap.

### Unmanaged 🔧

Take this C++ example:
```cpp
{
    std::string s = new std::string("Hello World");
    std::cout << *s << std::endl;
}
```
Here, the string `s` was created on the heap via the `new` operator. But what happens at the end of the scope, after the string is printed? Well, we have a leak...the memory is not freed. Although we never use the string `s` again, because it was created with the `new` operator, we must use the `delete` operator to free that memory after we are done. If we don't, that used memory will never be freed. This will lead to our application hogging more and more RAM, until we use it all! Hence, why C++ is called "unsafe".

Here's the safe C++ example:
```cpp
{
    std::string s = new std::string("Hello World");
    std::cout << *s << std::endl;
    delete s;
}
```
After its use, the string `s` is deleted and its memory is freed back to the system.

### Managed 🔧

Now, where does C# come into the picture? C# is a managed language, which means memory is cleaned up for us, automatically, via the garbage collector.

Take this safe C# example:
```csharp
{
    var s = new string("Hello World");
    Console.WriteLine(s);
}
```
Here, we see the same `new` operator as C++ used in creating the string `s` (meaning `s` is stored on the heap). But, after the string is printed, there is no `delete` operator used. This is because in C#, and other managed languages, the memory created on the heap is automatically freed. Therefore, C# is considered "safe" because we don't have to worry about memory leaks and deleting memory on the heap.

### You Can't Always Control The GC ⌛

If managed languages clean up all the memory for us, why doesn't everyone use a managed language? The main issue with languages that provide garbage collection (GC) is that you can't control when the GC decides to free the memory. Almost 99% of the time, the memory is freed after the object has been used (if freed before, you did something wrong), but how long after is uncontrollable. 

The nice part of unmanaged languages is that you the programmer decide when memory is created and deleted and have the most control. Sometimes this is critical for high-performance applications, such as games and trading software, which is why they normal stick to C++. However for the majority of applications, the "When will the GC kick in?" question isn't often asked and hence managed languages get the job done.

## [Issue 208](https://github.com/nlogozzo/NickvisionMoney/issues/208) 🪲

Issue 208 becomes the one exception of where the question: "When will the GC kick in?" is asked by us developers of Denaro 🙃. As described in the issue:
> I have also noticed a RAM usage issue that could be related: when adding a transaction or adding and removing a filter, the amount of RAM used increases and does not decrease. For my large database each addition of a transaction addition increases the RAM usage 200 MB, like it was loading all the database again and not removing the old one and keeps accumulating indefinitely. When opening the app and database, the RAM usage is around 300 MB and I have got to 3 GB of RAM usage by the app.

Right off the bat we know that this is a memory leak issue (memory is accumulating and not being freed). But, this is C#, memory is managed for us...so what's going on? Well, as mentioned before this is a case of "When will the GC kick in?". 

Previous versions of Denaro (when it was called Money 😉) handled any change to an account by redrawing the entire account. For example, if you created a transaction or changed a filter, all previous transaction rows were removed and then recreated and redrawn based off the new account information. Now for normal use cases of accounts with 100-200 transactions, this was totally ok. Computers are fast, C# is fast, 200 rows being redrawn for every change, although it doesn't sound like it, was fast and memory usage didn't spike. But, looking back at previous versions, the memory increase was there, but wasn't alarming (135 MB -> 145 MB). It wasn't until [@Suork](https://github.com/Suork) brought to our attention that Denaro is eating up all their RAM with an account that they have of 1200+ transactions. Those fast 200 redraws became slow 1400 redraws and huge increases in memory usage for any minor change, with no end in site.

### Identifying The Leak 🔍

Now keep in mind that [@Suork](https://github.com/Suork) was using the GNOME version of the app, but the same leak was present on WinUI as well. This made us believe that the leak was coming from the shared backend code of both platform versions (i.e. a controller or the account model), but we were wrong. After further investigation, we saw that memory was spiking when drawing rows for 400/500/600+ transactions. This was surprising to us for two reasons:
 1. This was happening on both UI platforms - GNOME and WinUI
 2. We were removing all previous rows before drawing the new ones 
    - Meaning the previous rows should have been GC'ed and the memory usage should stay relatively the same

For reasons still unknown, in both UI platforms, the old previous transaction rows, although removed from their list containers, were not being garbage collected. And because of this, every time information was changed both old and new transaction rows remained in memory, with no way of cleaning the old ones up (hence the 1+ GB of memory being used by Denaro).

### What To Do? 🤷‍♂️

The $1,000,000 question: "How do we fix this?"

Obviously, the old way of regenerating new transaction rows every time information in an account changes is not going to work. We thought it would be best to create a new transaction row at most one time and:
- Update the row directly when a transaction is updated
- Show/Hide the row when a filter is changed
- Move the row to a different position directly when sorting settings are changed

The problem arises that Denaro is available on two separate UI platforms and all of the logic described above is handled by the shared controller. Therefore, we need a way for the controller to be able to update, show/hide, and move the transactions rows of both platforms without using any UI specific code. Luckily for us, C# has interfaces.

## Interfaces 👨‍💻

Interfaces give us a way to ensure that certain objects contain the methods and properties defined by the interface. They are essentially "abstract classes" and because of this, interfaces cannot be instantiated as objects.

Take this interface below:
```csharp
public interface IAdder
{
    public int Add(int a, int b);
}
```
This is an interface that defines an `Add` method (notice how the method is not implemented). Because it is an interface, I cannot instantiate it, meaning `var adder = new IAdder()` will not work.

However, another class can extend the interface and that class can then be instated.

Take the following example:
```csharp
public class Calculator : IAdder, ISubtractor, IMultiplier, IDivider
{
    public int Add(int a, int b) => a + b;

    //other methods missing
}
```
Here, the `Calculator` class inherited the `IAdder` interface and implemented the `Add` method that is required. Because the `Calculator` class is not an `interface` it can be instantiated, meaning `var calc = new Calculator()` will work as normal.

### Upcasting ⬆️

Just because an interface cannot be instantiated does not mean that we cannot use it as a parameter. 

This technique is known as upcasting:
```csharp
IAdder calc = new Calculator();
```
Here we see `calc` is of type `IAdder` but it is being instantiated as a `Calculator` (which is valid since Calculator is not an interface). The `Calculator`, because it is upcasted to an `IAdder`, is restricted to only use the methods of `IAdder`. So even if Calculator had other methods such as `Sub`, `Mult`, etc..., because it is an `IAdder` on the left `calc.Add(..., ...)` is the only valid method we could use.

*Of course this is only valid if the class you are using to instantiate inherits the interface, otherwise your program  will not run.*

Because of upcasting, we can also use interface types as parameters to a function.
Take this function:
```csharp
public static int Add(IAdder adder, int a, int b) => adder.Add(a, b);

Console.WriteLine(Add(new Calculator(), 5, 3)); //prints 8
```
If we had a different class that inherited `IAdder` we could pass that instead of `Calculator` and still get the same result since, via the `IAdder` contract, that object will have an `Add` method.

## Interfaces In Denaro 🪙

Back in Denaro, we created interfaces for the row controls that we will have in both GNOME and WinUI.

### IModelRowControl<T> 🛂

The first interface is `IModelRowControl<T>`. This interface is generic as it allows for any model to be used for the type of row. In production, we will create `IModelRowControl<Transaction>` for transaction rows and `IModelRowControl<Group>` for group rows.

Here's the implementation:
```csharp
using System;

namespace NickvisionMoney.Shared.Controls;

/// <summary>
/// A contract for a model row control
/// </summary>
/// <typeparam name="T">The model type</typeparam>
public interface IModelRowControl<T>
{
    /// <summary>
    /// The Id of the model T the row represents
    /// </summary>
    public uint Id { get; }

    /// <summary>
    /// Occurs when the edit button on the row is clicked 
    /// </summary>
    public event EventHandler<uint>? EditTriggered;
    /// <summary>
    /// Occurs when the delete button on the row is clicked 
    /// </summary>
    public event EventHandler<uint>? DeleteTriggered;

    /// <summary>
    /// Shows the row
    /// </summary>
    public void Show();
    
    /// <summary>
    /// Hides the row
    /// </summary>
    public void Hide();

    /// <summary>
    /// Updates the row based on the new model
    /// </summary>
    /// <param name="newModel">The new model T</param>
    public void UpdateRow(T newModel);
}
```
This interface requires classes that inherit it to define an `Id` property, an `EditTriggered` event, a `DeleteTriggered` event, a `Show` method, a `Hide` method, and an `UpdateRow` method. 
We can now use this interface for our implementation of the `TransactionRow` control in the [GNOME](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.GNOME/Controls/TransactionRow.cs) and [WinUI](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.WinUI/Controls/TransactionRow.xaml.cs) projects respectively.

### IGroupRowControl 🛂

The second interface is `IGroupRowControl`. This interface is a specialization of `IModelRowControl` for the `Group` model. This is needed because groups can be filtered and thus need their own property and events to track that.

Here's the implementation:
```csharp
using NickvisionMoney.Shared.Models;
using System;

namespace NickvisionMoney.Shared.Controls;

/// <summary>
/// A contract for a group row control
/// </summary>
public interface IGroupRowControl : IModelRowControl<Group>
{
    /// <summary>
    /// Whether or not the filter checkbox is checked
    /// </summary>
    public bool FilterChecked { get; set; }

    /// <summary>
    /// Occurs when the filter checkbox is changed on the row
    /// </summary>
    public event EventHandler<(uint Id, bool Filter)>? FilterChanged;

    /// <summary>
    /// Updates the row based on the new Group model
    /// </summary>
    /// <param name="group">The new Group model</param>
    void IModelRowControl<Group>.UpdateRow(Group group) => UpdateRow(group, true);

    /// <summary>
    /// Updates the row based on the new Group model
    /// </summary>
    /// <param name="group">The new Group model</param>
    /// <param name="filterActive">Whether or not the filter checkbox is active</param>
    public void UpdateRow(Group group, bool filterActive);
}
```
Notice how this interface extends `IModelRowControl<Group>`, allowing us to just add the things we need and not rewrite everything from before. Classes and other interfaces can extend as many interfaces as they would like.
We can now use this interface for our implementation of the `GroupRow` control in the [GNOME](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.GNOME/Controls/GroupRow.cs) and [WinUI](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.WinUI/Controls/GroupRow.xaml.cs) projects respectively.

## Controller Management Of Rows 🎮

Now that we have our interfaces and implementation of our controls we can start moving management of these UI rows to the `AccountViewController` and away from the `AccountView` itself. Remember all the logic of filtering, sorting, and creating/updating models is all done from the controller. Moving this to the `AccountView` itself would require us to copy all this logic twice: once for GNOME and once for WinUI. So instead, we are moving row management to the controller where all the code is written once and will work automatically for both platforms because of the interfaces we made before.

### Properties 🔧

In our controller we will define some properties:
```csharp
/// <summary>
/// The list of UI transaction row objects
/// </summary>
public Dictionary<uint, IModelRowControl<Transaction>> TransactionRows { get; init; }
/// <summary>
/// The list of UI group row objects
/// </summary>
public Dictionary<uint, IGroupRowControl> GroupRows { get; init; }
/// <summary>
/// The UI function for creating a group row
/// </summary>
public Func<Group, int?, IGroupRowControl>? UICreateGroupRow { get; set; }
/// <summary>
/// The UI function for deleting a group row
/// </summary>
public Action<IGroupRowControl>? UIDeleteGroupRow { get; set; }
/// <summary>
/// The UI function for creating a transaction row
/// </summary>
public Func<Transaction, int?, IModelRowControl<Transaction>>? UICreateTransactionRow { get; set; }
/// <summary>
/// The UI function for moving a transaction row
/// </summary>
public Action<IModelRowControl<Transaction>, int>? UIMoveTransactionRow { get; set; }
/// <summary>
/// The UI function for deleting a transaction rowe
/// </summary>
public Action<IModelRowControl<Transaction>>? UIDeleteTransactionRow { get; set; }
```
The first two dictionaries will be the lists of our rows. The key will by the id of the model for easy reference. The next five properties are function objects for creating, moving, and deleting rows via the UI. Remember that the shared project, where the controller lives, has no knowledge of UI platforms and thus has no way of creating a row and adding it to a list box from the controller. Therefore, the `AccountView` of each respective platform will assign these properties to their respective functions.

### Account View Implementations 👨‍💻

Let's take a look at the [GNOME implementation](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.GNOME/Views/AccountView.cs) implementation:
```csharp
namespace NickvisionMoney.GNOME.Views;

public partial class AccountView
{
    ...

    /// <summary>
    /// Constructs an AccountView
    /// </summary>
    /// <param name="controller">AccountViewController</param>
    /// ...
    public AccountView(AccountViewController controller, ...)
    {
        ...
        //Register Controller Events
        _controller.AccountTransactionsChanged += OnAccountTransactionsChanged;
        _controller.UICreateGroupRow = CreateGroupRow;
        _controller.UIDeleteGroupRow = DeleteGroupRow;
        _controller.UICreateTransactionRow = CreateTransactionRow;
        _controller.UIMoveTransactionRow = MoveTransactionRow;
        _controller.UIDeleteTransactionRow = DeleteTransactionRow;
        ...
    }

    /// <summary>
    /// Creates a transaction row and adds it to the view
    /// </summary>
    /// <param name="transaction">The Transaction model</param>
    /// <param name="index">The optional index to insert</param>
    /// <returns>The IModelRowControl<Transaction></returns>
    private IModelRowControl<Transaction> CreateTransactionRow(Transaction transaction, int? index)
    {
        var row = new TransactionRow(transaction, _controller.CultureForNumberString, _controller.Localizer);
        row.EditTriggered += EditTransaction;
        row.DeleteTriggered += DeleteTransaction;
        row.IsSmall = _parentWindow.DefaultWidth < 450;
        if(index != null)
        {
            _flowBox.Insert(row, index.Value);
            g_main_context_iteration(g_main_context_default(), false);
            row.Container = _flowBox.GetChildAtIndex(index.Value);
        }
        else
        {
            _flowBox.Append(row);
            g_main_context_iteration(g_main_context_default(), false);
            row.Container = _flowBox.GetChildAtIndex(_controller.TransactionRows.Count);
        }
        return row;
    }

    ...
}
```
Let's take zoom in on `AccountView.CreateTransactionRow`. The first thing to notice is the method header:
```csharp
private IModelRowControl<Transaction> CreateTransactionRow(Transaction transaction, int? index)
```
This comes from the definition of the UICreateTransactionRow property in the controller, which tells the `AccountView` how it should define it's function:
```csharp
public Func<Transaction, int?, IModelRowControl<Transaction>>? UICreateTransactionRow { get; set; }
```
The `Func` part means that all the parameters in the `<>` brackets will be arguments to the function, except the last one which will be used for the return type. Hence why in `CreateTransactionRow` it takes `Transaction` and `int?` for arguments and returns `IModelRowControl<Transaction>`.

Now, notice how inside the `CreateTransactionRow` it creates its platform's specific version of `TransactionRow` and uses some `GTK` specific functions to draw the row to the `Gtk.FlowBox`, etc..., etc...

The important thing to note is that in the end it simply returns `row`. This is possible because in the [GNOME implementation](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.GNOME/Controls/TransactionRow.cs) of the `TransactionRow`, the `IModelRowControl<Transaction>` interface is inherited and implemented. This allows the `TransactionRow row` variable to simply be returned as `IModelRowControl<Transaction>` thanks to upcasting (previously described above).

The [WinUI implementation](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.WinUI/Views/AccountView.xaml.cs) of `AccountView`, of course, works exactly in the same way.

The power of upcasting allows us to create platform UI controls and abstract that all away to the interface that will be used in the controller, without knowledge of UI platform specific implementations.

### Controller Work 👷‍♂️

Ok, so we now have properties defined and `AccountView` functions wired up. How do we use this all in the `AccountViewController` itself?

Let's take a look at when a transaction is added via the [controller](https://github.com/nlogozzo/NickvisionMoney/blob/main/NickvisionMoney.Shared/Controllers/AccountViewController.cs):
```csharp
/// <summary>
/// Adds a transaction to the account
/// </summary>
/// <param name="transaction">The transaction to add</param>
public async Task AddTransactionAsync(Transaction transaction)
{
    var groupId = transaction.GroupId == -1 ? 0u : (uint)transaction.GroupId;
    await _account.AddTransactionAsync(transaction);
    var transactions = _account.Transactions.Keys.ToList();
    transactions.Sort((a, b) =>
    {
        var compareTo = SortTransactionsBy == SortBy.Date ? _account.Transactions[a].Date.CompareTo(_account.Transactions[b].Date) : a.CompareTo(b);
        if (!SortFirstToLast)
        {
            compareTo *= -1;
        }
        return compareTo;
    });
    for(var i = 0; i < transactions.Count; i++)
    {
        if (transactions[i] == transaction.Id)
        {
            TransactionRows.Add(transaction.Id, UICreateTransactionRow!(transaction, i));
        }
        if (_account.Transactions[transactions[i]].RepeatFrom == transaction.Id)
        {
            TransactionRows.Add(_account.Transactions[transactions[i]].Id, UICreateTransactionRow!(_account.Transactions[transactions[i]], i));
        }
    }
    GroupRows[groupId].UpdateRow(_account.Groups[groupId]);
    FilterUIUpdate();
}
```
The important line to focus on is:
```csharp
TransactionRows.Add(transaction.Id, UICreateTransactionRow!(transaction, i));
```
Here, we are adding a new value to the `TransactionRows` dictionary with the key being the transaction's id and the value being what's returned by `UICreateTransactionRow` (which from the properties above we know is a `IModelRowControl<Transaction>`). Although the controller doesn't know how `UICreateTransactionRow` is implemented, since it was defined in the `AccountView` (as seen above), we know it will have created a row and displayed it on the UI and return to us the upcasted `IModelRowControl<Transaction>` version of it.

In this way, when we add a transaction, only one singular row is created. Before, all the rows would have been removed, recreated, and redrawn.

This same technique is then used when a transaction is updated, deleted, etc... 

Only the rows that need to change, change. Everything else remains untouched. 

**No unnecessary redrawing, no unnecessary recreating, no memory usage spike.**

<script src="https://utteranc.es/client.js"
        repo="nlogozzo/nlogozzo.github.io"
        issue-term="url"
        label="blog-comments"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>