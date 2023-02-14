# Using-lazy-properties-in-swift

Lazy properties allow you to create certain parts of a Swift type when needed, rather than doing it as part of its initialization process. This can be useful in order to avoid optionals, or to improve performance when certain properties might be expensive to create. It can also help with keeping initializers more clean, since you can defer some of the setup of your types until later in their lifecycle.

This week, let’s take a look at a few ways to define lazy properties in Swift, and how different techniques are useful in different situations.

# The basics

The easiest way to define a lazy property, is to simply prefix the var keyword with lazy, and give it a default value. This default value will be assigned as soon as the property is accessed, meaning that it won’t necessarily be evaluated during the type’s initialization process.

```swift
class FileLoader {
    private lazy var cache = Cache<File>()

    func loadFile(named name: String) throws -> File {
        // The first time this code is run, the cache will be lazily created
        if let cachedFile = cache[name] {
            return cachedFile
        }

        let file = try loadFileFromDisk(fileName: name)
        cache[name] = file
        return file
    }
}
```

# Using a factory method 

However, sometimes you need to perform a bit more setup of the object that is being lazily created, and simply using its initializer might not be enough. For such situations, delegating the creation to a factory method is a nice way to give you a lot more flexibility:

```swift 
class Scene {
    private lazy var eventManager = makeEventManager()

    func add(event: Event) {
        // As soon as we add an event, the manager will be created and started
        eventManager.register(event)
    }

    private func makeEventManager() -> EventManager {
        let manager = EventManager()
        manager.startLoop(for: self)
        return manager
    }
}
```

Note that if you're using Swift 3 or earlier, you have to add an explicit type and a self reference to the above lazy property:

```swift
private lazy var eventManager: EventManager = self.makeEventManager()
```

In case you don’t want to fill up your classes with a lot of make…() methods, you can always move them to a dedicated private extension.

# Using a self-executing closure

Instead of creating a new method for every lazy property, you can choose to inline your setup code in the property declaration itself, using a self-executing closure. Let’s take a look at how the above example would look if we would use a self-executing closure for eventManager instead of a method:

```swift
class Scene {
    private lazy var eventManager: EventManager = {
        let manager = EventManager()
        manager.startLoop(for: self)
        return manager
    }()

    func add(event: Event) {
        eventManager.register(event)
    }
}
```

The advantage of this approach is that you can keep both the property declaration and its setup in one place, but the downside is that it can lead to pretty hard to read code if the setup is quite lengthy (since it “pushes down” the other content of your type). My rule of thumb is therefor to use this approach only when performing simple setup (like 3-4 lines maximum).

# Using a static factory

Sometimes it’s a good idea to move the setup code of some more complex properties out from their type. That way, you can maintain a more clear area of responsibility for your types, and reduce their complexity. It’s also a nice way to share setup code between multiple types without having to use subclassing.

Let’s take a look at an example where a view controller needs an action button once an info view has been presented, and instead of setting up the action button inline, we use a static factory:

```swift
class ViewController: UIViewController {
    private lazy var actionButton = ViewFactory.makeActionButton()

    func present(infoView: InfoView) {
        view.addSubview(infoView)
        infoView.frame = view.bounds.offsetBy(dx: 0, dy: view.bounds.height)
        infoView.present(animated: true)

        // The action button will be created here if needed
        actionButton.center = view.center
        view.addSubview(actionButton)
    }
}
```

Now, ViewFactory can contain all the setup code for our shared custom views, without having to introduce a lot of new types or rely on a complex inheritance tree. If we need to create an action button in another view controller, we can just use the same ViewFactory.makeActionButton() API to do so.
