<img  src="https://raw.githubusercontent.com/IanSSenne/mcbuild/master/assets/MCB%20Title%20B.png"/>

# mc-build
mc-build is a cli tool that helps with the creation of data packs through compiling a custom format to functions.

## Installation
You can watch this excellent video created by AncientKingg for a tutorial on how to install MCB:
https://streamable.com/bcljw4

### yarn install command
```bash
$ yarn global add mc-build
```
### npm install command

```bash
$ npm i -g mc-build
```

## CLI
|command | result|
|--------|-------|
|`mcb` | will build the project in the active directory|
|`mcb -config [js,json]` | generate a config file based on the default configs of all loaded languages. |

## Setting up a MCB Project
Ok. so you've installed MCB using the tutorial above, what next?
It's time to setup a MCB project! Don't worry, it's quite similar to how you setup a normal datapack
1. Create a world in Minecraft
2. Open the datapacks folder (`.minecraft/saves/world_name/datapacks/`) and create a folder, name this folder whatever you want within reason.
3. Inside the folder you just created, and make a folder called `src`
4. Inside of `src` create a file called `namespace.mc` (You can replace `namespace` with whatever you want. It will be the namespace you use when calling a function declared inside of that .mc file)
5. Then, open Command Line and navigate to your datapack's folder (The one with `src` in it) and run `mcb`
6. Your project is now ready to go!


## The MC Language
The default language that ships with MCB

#### Function Definition
You can define a basic mcfunction using the `function` keyword:

```
function example_function {
	say hello, this is a function
}
```

## Compile Time Logic
In this section we will talk about the compile time logic MCB has to offer.

#### JS blocks
You can inject JS directly into commands during run time using the in-line JS block.
For instance if I wrote a function like this:
```
function example {
	say <%config.dev%>
}
```
and I set the value of `dev` in the config to `"Bob's your uncle"`, when the project compiles the function would look like this:
```
say Bob's your uncle
```
There are also multiline JS blocks, which allow you to run entire scripts and `emit` their results into a function!
In order to `emit` code into a file you use the method `emit()` within a JS multi-line block. Note that this method does not exist in an in-line JS block. Unless the `emit()` method is used a multi-line JS block will not add or change anything inside your function. Each block is it's own private environment, so variables will not be saved outside of the block with the exception of `global` variables (More on that later)
```
<%%
	console.log("This will print on compile time!");
	emit('say This will appear as code in the function!')
%%>
```

---
#### Compile time IF statement
The compile time if statement allows you to decide whether to include or exclude code in the compiled result based off a variable in the config. Very useful for adding/removing debugging tools when building in-dev or release versions. 
```
function example {
	!IF(config.dev){
		say This code will only be placed in this function IF value of dev is truthy in your mcb config on compile time.
	}
	say this code is always placed in the function
}
```
There is also a shorthand version of the Compile time if that works the same, but is easier to type:
```
function example {
	!config.dev{
		say This code will only be placed in this function IF value of dev is truthy in your mcb config on compile time.
	}
	say this code is always placed in the function
}
```
---
#### Compile Time Loop
The compile time loop allows you to duplicate code based on a value on compile time. The loop will pass it's current iteration to `this.i` inside of in-line JS blocks.
In this example we take the current iteration and put it's value into a `say` command using an in-line JS block:
```
function example {
	LOOP(5,i){
		say <%this.i%>
	}
}
```
When compiled this function would look like this:
```
say 0
say 1
say 2
say 3
say 4
```
---
#### Execute Run Block
An Execute Block allows you to define a function after an execute run statement
```
function example {
	execute as @a run{
		say hi
	}
}
```
This would compile to
```
execute as @a run function namespace:__generated__/execute/0.mcfunction
```
in this case, the generated function would just contain `say hi`.

---
#### Inline Blocks
The Inline keyword allows you to define a function within another function. When compiled the inline block produces a new function, and inlines a call to that function
```
function example {
	say Outside of inline block!
	inline{
		say Inside of inline block!
	}
}
```
would compile to
```
say Outside of inline block!
function namespace:__generated__/inline/0.mcfunction
```
with the generated function containing `say Inside of inline block!`
This is slightly different from an execute run block, where if you were to try something like this:
```
execute run{
	say hi
}
```
it would compile to
```
execute run function ...
```
which is not an acceptable practice according to most command devs, however Minecraft will just omit the `execute run` command because it's smart. So it's not something we recommend doing, but it won't effect anything internally except for possibly file size and perhaps your reputation among the MCC community.

---
#### dir blocks
Using dir blocks You can define sub-folders inside of your .mc file.
```
dir test{
	function example {
		say hi
	}
}
```
This would put the function `example` under this path: `namespace:test/example`
Dir blocks can be nested to create sub folders of sub folders.

## Runtime Logic

#### Runtime Execute If-ElseIf-Else Blocks
Runtime execute if blocks allow us to choose between a number of functions based on execute conditions during runtime! The command code behind this is very well optimized and should not effect performance.
```
function example {
	execute(if score #count value matches 0){
		say 0
	}else execute(if score #count value matches 1){
		say 1
	}else{
		say not 0 or 1
	}
}
```
This is basically a shorthand way of writing this:
```
scoreboard players set #MCBIF <%this.internalScoreboard%> 0
execute if score #count value matches 0 run{
	scoreboard players set #MCBIF <%this.internalScoreboard%> 1
}
execute if score #MCBIF <%this.internalScoreboard%> matches 0 if score #count value matches 1 run{
	say 1
	scoreboard players set #MCBIF <%this.internalScoreboard%> 1
}
execute if score #MCBIF <%this.internalScoreboard%> matches 0 run{
	say not 0 or 1
}
```
---

#### Wait Block
`wait(condition,poll_rate)` will check the `condition` every `poll_rate` and if that condition is true, it will run it's block.
`condition` is an execute sub-command chain.
`poll_rate` is a schedule command delay, eg: `1t`, `24d`, `2s`, etc...

In this example we are checking if there are any players with a score of `deaths` greater than 10, and if one is found we tell them they have died at least 10 times. This will only happen for the first player over 10 deaths as the wait loop will exit once the condition is met
```
function example {
	wait(as @a[scores={deaths=10..},limit=1], 1s){
		tellraw @s "You have died at least 10 times"
	}
}
```
Keep in mind that the wait block loses context of it's parent function due to the nature of the `schedule` command (which is used to check the condition based on a poll rate)

---

