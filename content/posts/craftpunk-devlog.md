---
title: "Making a modular crafting system in Godot"
date: 2021-11-03T21:13:13-04:00
draft: false
summary: "Extend Godot resources to create a modular crafting system."
description: "Craftpunk devlog: extending Godot resources to create a modular crafting system."
feature_image: '/images/laserpunk-logo.png'
category: "devlog"
tags: [ "Craftpunk", "devlog" ]
---


![Crafting System Demo](/images/craftpunk-crafting-demo.gif)


## Godot Resources

Godot's [Resource](https://docs.godotengine.org/en/stable/getting_started/step_by_step/resources.html) class is a serializable data container that can be extended. They are quite similar in usage to Unity's ScriptableObject class. The following are some use cases and strengths of using Resources in your game project.

- Resources expose properties to the editor. This allows game designers to create new objects of the new Resource type in the editor.
- Resources are serializable, so they can be easily saved and loaded from disk.
- Resources are more light-weight than Nodes, and help streamline the SceneTree.

These all play a role in keeping the delineations of the various game systems well-defined.

- Pure data such as player stats are stored and manipulated in Resources.
- Nodes (e.g. actors, UI) can fetch data and perform abstract operations on the Resource.


## Crafting

It is fairly trivial to apply Resources to an inventory system where items are static data objects: one inventory item maps to one base class that derives from Resource. For [Craftpunk](https://juniper-dusk.itch.io/craftpunk), I followed the [Resource Based Inventory](https://www.youtube.com/watch?v=rdUgf6r7w2Q) tutorial by Heartbeast. I have yet to go back and refactor this system, but I will make a note of where improvements can be made.


### Base Class
```
# craft_component.gd
class_name CraftComponent
extends Resource


export(Texture) var thumbnail
export(String) var name
export(int) var amount = 1
export(bool) var stackable = true


func get_description() -> String:
    return "Raw material"
```

From there, you can start to create whatever raw materials (e.g. iron ore, copper ore) by creating a new `CraftComponent` in the editor and filling out the properties.


### Crafting Recipes
```
# craft_recipe.gd
class_name CraftRecipe
extends CraftComponent


export(Array, Resource) var components
var _initialized : bool


func get_description() -> String:
    return "Compound material"


static func get_material_amounts(list : Array) -> Dictionary:
    # Generate name/amount dictionary.
    var result = {}
    for material in list:
        if not material.name in result:
            result[material.name] = material.amount
        else:
            result[material.name] += material.amount

    return result
```

Overall, a recipe is fairly simple: it is a list of items. Because `CraftRecipe` inherits from the base item class, recipes can compose other recipes and be stored in the inventory.

I opted for a simpler approach on the front-end, where the designer simply adds duplicate Resources to represent more of that object. I find the exported Dictionary interface to be a bit finicky compared to [exported Arrays](https://docs.godotengine.org/en/stable/getting_started/scripting/gdscript/gdscript_exports.html#exporting-arrays), because it lacks type hinting; hopefully this changes in a future update.

This does have the drawback of some added complexity on the backend: we need to iterate over the list and generate a dictionary with unique items as keys, and amounts as values. *NOTE: Ideally, you would store this data in the CraftRecipe upon initialization, so that accessors do not need to know of this function.*

### Recipe Database
```
# recipe_database.gd
class_name RecipeDatabase
extends Resource


export(Array, Resource) var recipe_list : Array
var recipe_database : Dictionary


func initialize() -> void:
    # Construct recipe database.
    recipe_database = {}
    for recipe in recipe_list:
        recipe_database[recipe.name] = CraftRecipe.get_material_amounts(recipe.components)


# Param: materials is a dictionary with component name/amount as key/value.
func get_recipe(materials : Dictionary) -> CraftComponent:
    for recipe_name in recipe_database.keys():
        var recipe_amounts = recipe_database[recipe_name]
        if materials_match_recipe(recipe_amounts, materials):
            return get_component_by_name(recipe_name)

    return null


func materials_match_recipe(recipe : Dictionary, materials : Dictionary) -> bool:
    # Don't attempt to read from an empty materials list.
    if not materials.keys():
        return false

    for name in recipe.keys():
        if not name in materials:
            return false
        elif materials[name] < recipe[name]:
            return false

    for name in materials.keys():
        if not name in recipe:
            return false

    return true


func get_component_by_name(name : String) -> CraftComponent:
    for recipe in recipe_list:
        if recipe.name == name:
            return recipe
    return null
```

Once you create a number of recipes, you need to query a collection of them in order to match a "pool" of items to an existing recipe. In a larger-scale project, this could be a NoSQL database which would allow for storage of many items and allow for server-side replication. However, for time's sake I went with a relatively simple Dictionary wrapper.

One benefit of storing the list in this way is that recipes can be "learned" by the player as the game progresses, and this can be serialized and saved to disk.

The main complication here is that items need to be fetched by the unique `name` field, whereas in memory they are stored as Resources which do not have a extendable hash function. This means we lose the instant access from the hash table and have to iterate over the keys linearly like an Array. I kept it this way to eventually split it out into C++ and override the hash operator. *NOTE: You could store a separate name -> Resource mapping alongside the name -> materials mapping.*


## Inventory

### Craft Pool
```
# craft_pool.gd
class_name CraftPool
extends ItemPool

export(Resource) var recipe_database

func initialize():
    recipe_database.initialize()

func get_item() -> CraftComponent:
    # Tally up number of materials by name.
    var material_amounts = CraftRecipe.get_material_amounts(materials)
    # Verify with database if item can be crafted.
    return recipe_database.get_recipe(material_amounts)

# Sets materials to the leftover materials.
func craft_item() -> bool:
    # Make sure we have an item to begin with.
    var crafted_item = get_item()
    if not crafted_item:
        return false
    crafted_item = crafted_item.duplicate()

    var remaining_amounts = CraftRecipe.get_material_amounts(materials)
    var count = 0
    var item = crafted_item.duplicate()
    var item_amounts = CraftRecipe.get_material_amounts(item.components)

    # While we have an item, keep subtracting required resources.
    while(item):
        if subtract_amounts(remaining_amounts, item_amounts):
            # Enough materials left, add one to count.
            count += 1
            item = recipe_database.get_recipe(remaining_amounts)
        else:
           item = null

    # Reset materials to clear pool.
    reset()

    # Add leftover materials from crafting.
    for name in remaining_amounts.keys():
        var new_item = recipe_database.get_component_by_name(name).duplicate()
        new_item.amount = remaining_amounts[name]
        add_material(new_item)

    # Add the count to the crafted item and append it to list.
    if crafted_item.stackable:
        crafted_item.amount = count
        add_material(crafted_item)
    else:
        for _i in range(count):
            add_material(crafted_item)

    return true
```

Now that we have our crafting system essentially complete, we just need a way to bridge the gap between our inventory and the crafting system. I chose to keep the two systems fairly encapsulated from each other and copy items to the `CraftPool` when the craft button is pressed. You could extend the actual `Inventory` resource from the HeartBeast tutorial to avoid copying, but that class is complex enough as is in my opinion.

This class is essentially responsible for processing "batches" of recipes. It verifies the ingredients are present, crafts the corresponding item and subtracts the necessary amounts.


## Conclusion

Once this is all done, one simply needs to extend the Inventory class from the earlier tutorial. Connect some sort of button to a signal and trigger the following:
- Copy over the materials to the `CraftPool`.
- Call the `CraftPool.craft_item()` function.
- If anything is returned, set the items in the inventory to the return value.

I will release the full code samples for the crafting system in a public repository soon and post the link here.

### Exercises

To improve upon this system, here are some suggestions:
- Each recipe could have a "time" property which kicks off a wait timer in the inventory.
- Add an actual database to store the items and recipes in the code, preferably a NoSQL one like MongoDB.