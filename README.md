# Title

Either approaches present some pros and cons.

- Callbacks are **much more flexible since you can append additional logic after loading is complete**, as well as callbacks **for error handling or updating**.

- However, they present some **challenges regarding thread and memory safety**. Not only does there need to be logic for handling the callbacks themselves **synchronously**, but if the caller is moved (not the handle), the **captured context of the callback is invalidated**.

- Assigning the handles and then retroactively checking is easier to implement and less likely to 

- However, the user of said handle **must check it for validity everytime it is used**, as the flag (which should be atomic), is the only thing stopping race conditions from happening.

For my use case however, I chose to go with assigning the handles the requested data and have my user check it everytime it needs to be used.

I chose this option initially not only because **binding callbacks can potentially create very unsafe code if I do not implement it correctly**, but also because I want my project to work with bee, which does work with an **ECS that can't guarantee pointer stability**. 

### Week 4

After my feedback session with Bert, he told me that this was very relevant problem in the world of **web programming and connection / request handling of servers**.

This made me reconsider the **callback approach in a couple ways to my system**. Looking more carefully at the previous examples I provided, Unity does provide a way to bind a callback to the completion of a request.

The trick is that the callback **should only work on the asset itself to execute some initialization logic** and **not capture any outside variable when the asset was requested**.

This **will still force my user to check for the handle every time** it is needed (which probably is not even too much of a performance burden in reality), but it will **allow me to execute an asynchronous callback when the loading is done** (which is very useful to queue up all the textures and models for upload in OpenGL)

## How to accurately count memory usage for specific assets: