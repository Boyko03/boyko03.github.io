---
layout: post
title:  "How to generate Lua scripts from a Node Graph based visual script?"
date:   2024-01-26 12:44:01 +0100
categories: jekyll update
---

![Visual script](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/graph.png)

## How to generate Lua scripts from a Node Graph based visual script?

Visual scripting or visual programming is a form of programming that lets the user create programs by using graphical elements instead of writing traditional code. This makes it easier to use by non-programmers, as they don't have to learn a specific language, and allows for faster prototyping by designers.  
In this blogpost I will talk about how I made a tool for creating a visual script based on a node graph, and then generating a Lua script from it.

### Table of contents:

- [A tool for creating visual scripts](#a-tool-for-creating-visual-scripts)
    - [1. Structure of a visual script](#1-structure-of-a-visual-script)
    - [2. The graph and the nodes](#2-the-graph-and-the-nodes)
    - [3. Undo and Redo - The command pattern](#3-undo-and-redo---the-command-pattern)
- [Evaluating the graph and generating the code](#evaluating-the-graph-and-generating-the-code)
- [Resources](#resources)

### A tool for creating visual scripts

#### 1. Structure of a visual script

There are different approaches for creating visual scripts. The two most common ones are:
- The [Unreal Engine Blueprints](https://docs.unrealengine.com/5.3/en-US/blueprints-visual-scripting-in-unreal-engine/), where the blueprint contains both the object and the script.
- The [Unity Visual Scripting](https://unity.com/features/unity-visual-scripting), where the script is added to the game object as a component.

I decided to use the second approach because it gives the user the flexibility to add and remove script components at runtime, and because it pairs well with the entity component system (ECS).

My visual scripts consist of the following elements:
- Events that they implement (such as `OnTick` and `OnCollision`)
- Functions
- Variables
- Each event and function can have their own local variables.

![Script elements](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/ScriptElements.png)

The script variables can be shown in the inspector and they act like an interface for the user to use in order to customize the behaviour of the script.

![Script component shown in the inspector](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/ScriptInterface.png)

Adding a custom event is as simple as adding the name of the event to the map of events in the `config.hpp` and specifying the parameter names of that event and their value type:  
![Adding events](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/events_map.png)

#### 2. The graph and the nodes

Every function and event have their own node editor that stores their graph.  
As a library for showing the graph I chose to use [imnodes](https://github.com/Nelarius/imnodes), because it's lightweight, easy to use, has good documentation and examples, and provides a graph implementations with IdMaps so that the nodes and the edges are sorted.

A node has two representations:
- A `Node` that stores information about the node itself (note that pins are nodes in the graph too), the value that it has, and other information necessary for representing it as a Lua statement.
- A `UiNode` that stores information about the UI representation of the node, such as name, IDs of input pins, IDs of output pins, etc.

An example of the node editor showing a function:  
![Example function](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/function_body.png)

#### 3. Undo and Redo - The command pattern

In order to implement undo / redo for the tool, adding nodes had to be done using the command pattern.  
This means that in order to be able to add a new node to the graph, the command has to know how to build the node. This information is passed using the `NodeData` struct:

```cpp
struct InputPin
{
    std::string name;
    pin_color pin_color = pin_color_default;
    ValueType type = ValueType::nil;
    registered_types value = {};
    std::string op; // Used for enums
};

struct OutputPin
{
    std::string name;
    pin_color pin_color = pin_color_default;
};

typedef int NodeDataFlags;
enum NodeDataFlags_
{
    NodeDataFlags_None = 0,
    NodeDataFlags_HasExecutionPinIn = 1 << 0,     // Sets whether node has an execution pin in
    NodeDataFlags_HasBreakPin = 1 << 1,           // Sets whether node has a break pin
    NodeDataFlags_ConnectOutputPinToNode = 1 << 2 /* Make edges between node and output pins.
(Default behaviour is to make edges between output pins and node only)*/
};

struct NodeData
{
    ImVec2 click_pos;
    std::string name;
    std::string func_op;
    NodeType node_type;
    NodeEditor::UiNodeType ui_node_type;
    NodeEditor::node_types ui;
    std::vector<InputPin> inputs = {};
    std::vector<OutputPin> outputs = {};

    NodeDataFlags flags = NodeDataFlags_None;
    int num_execution_pin_out = 0;
};
```

Example usage - creating a binary operator node:  
![Create node](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/create_node.png)

In order to add new types of nodes (that generate Lua code in a different way than the already available ones), the user has to add a new value to `NodeType` enum and then implement a case in the `Node::evaluate` specific for that new type. The user can also make a new value to the `UiNodeType` enum if they want a custom representation of the node in the graph.  

**Make sure to always add new values to the bottom of the enum, because otherwise you are going to break existing visual scripts!**  
**Before adding a new value and implementation, check whether the new node can use one of the available types.**

### Evaluating the graph and generating the code

The algorithm for generating a Lua script is the following:
1. Create a new type in Lua with the name of the script.
- A table that is going to have the variables, functions, and events of the script.
- Used to create new instances of that script when emplacing it on an entity.
2. Generate constructor:  
- The arguments of the constructor are the variables of the visual script.  
3. Generate events and functions
4. Return the newly created type so that it can be used for creating new instances.

Algorithm for generating the body of a function:  
![Graph evaluation](/assets/posts/2024-01-26-generating-lua-scripts-from-a-node-graph-based-visual-script/node_evaluation.png)
1. Generating the body of a function starts from the `begin` node.  
2. The algorithm follows the `execution pin out` of the `begin` node and goes to the `execution pin in` to which it is connected.
3. The `execution pin in` is connected to a node.
4. Based on the type of the node, the algorithm decides how to evaluate its `input pins` and what Lua code to generate.
5. The `input pins` are connected to `output pins`.
6. The `output pins` are connected to a node and evaluate themselves based on the type of the node they are connected to.
7. If the node has an `execution pin out` the algorithm follows it towards the next node as in step 2.
8. Based on the type of the node, it can have multiple `execution pins out` or even some custom pins. In this case, the user has to specify how the node should be processed by the algorithm so that the code is generated correctly.
9. If the node isn't connected to an `execution pin out` pin or that pin isn't connected to an `execution pin in`, the algorithm stops and returns the generated Lua script.

After the algorithm is finished, the generated Lua script is saved and is added to the script manager class so that it can be added to entities as a component.

Example of a script generated by the tool:

```lua
script2 = {}
script2.__index = script2

function script2.new(entity, speed, GamepadID)
	local o = {}
	setmetatable(o, script2)

	o.entity = entity
	o.speed = speed
	o.GamepadID = GamepadID

	return o
end

function script2:OnTick(dt)
	local direction = vec2.new(0.000000, 0.000000)
	if 	IsGamepadAvailable(self.GamepadID) then
		self:move_with_gamepad(dt)
	else
		if 	GetKeyboardKey(KeyboardKey.W) then
		direction = (direction + vec2.new(0.000000, 1.000000))

	end

	if 	GetKeyboardKey(KeyboardKey.S) then
		direction = (direction + vec2.new(0.000000, -1.000000))

	end

	if 	GetKeyboardKey(KeyboardKey.A) then
		direction = (direction + vec2.new(-1.000000, 0.000000))

	end

	if 	GetKeyboardKey(KeyboardKey.D) then
		direction = (direction + vec2.new(1.000000, 0.000000))

	end

	GetBody(self.entity).position = (GetBody(self.entity).position + (direction * (dt * self.speed)))

	do return  end

	end

end

function script2:move_with_gamepad(dt)
	local body = nil
	body = GetBody(self.entity)
	body.position = ((vec2.new(	GetGamepadAxis(self.GamepadID,GamepadAxis.StickLeftX),(0.000000 - 	GetGamepadAxis(self.GamepadID,GamepadAxis.StickLeftY))) * (self.speed * dt)) + body.position)

	do return  end
end

return script2

```

<br>

---

<br>

To summarize, in this blogpost I talked about different ways of implementing visual scripting, the tool I made for creating a visual script based on a node graph, and then how does the algorithm that generates a Lua script from it work.  
I hope this article was interesting and informative to you.  

If you have any questions feel free to reach me.


### Resources:

[https://unity.com/features/unity-visual-scripting](https://unity.com/features/unity-visual-scripting)  
[https://docs.unrealengine.com/5.3/en-US/blueprints-visual-scripting-in-unreal-engine/](https://docs.unrealengine.com/5.3/en-US/blueprints-visual-scripting-in-unreal-engine/)

---

![BUas logo](https://www.buas.nl/sites/default/files/2018-09/Logo%20BUas_RGB.png)
