# Codecs:
The best part of Mojang's DataFixerUpper, the best serialization library on the market, made by Mojang themselves!
A codec allows for easy conversion from Java objects to serialized objects and vice versa.
This is very useful for config files and custom dynamic registries, but they're also the primary way to read and write nbt these days.
SavedData even forces you to set up everything as a codec these days.

DataFixerUpper includes conversion directly between Java code and Json elements.
Minecraft additionally includes NbtOps for working with NbtElements, HashOps for computing a hash code of anything with a codec and NullOps for data validation (you probably won't be using HashOps and NullOps very often, but it's worth knowing they exist). There's also something called RegistryOps which I'll get back to.

You can find codecs for the primitive data types in the Codec class, and a variety of additional codecs in the ExtraCodecs class.
We also have some extras in the `METACodecs` class in metacraft-lib, feel free to add more as necessary (or remove ones that are redundant, some of them are for Mojang classes that lack codecs at the moment).
Minecraft classes usually put their codecs within the class themselves, so you'll find the ItemStack codec in the ItemStack class for example.
Rule of thumb, if you can modify the class you want to add a codec for, put the codec inside that class. If you can't (because it's from a library or something), put it in `METACodecs` (or another class if the library is not visible to METAcraftLib).

Serializing data with codecs is easy, just take your codec and call `Codec::encodeStart` with your ops of choice to convert the value to the desired format. This returns a DataResult, which is literally just an object representing either your result, or an error message if something went wrong. When serializing, ideally nothing should go wrong so you can usually call `DataResult::getOrThrow`.
To deserialize an object, call `Codec::parse` with the ops matching the type you want to deserialize and you'll once again get a DataResult. In this case the data might be invalid, so I recommend calling `DataResult::resultOrPartial` (the one that takes a String supplier as an argument) with a relevant `LOGGER::error` as the argument. This way, the error will be logged so you can see it, but if some of the data is salvageable it will salvage it. When working with nbt, you don't even have to worry about this since they give you a put function which writes the value with the given codec, and a read function which reads the value with the given codec.

Note to actually write codecs to file you'll have to write the given data types manually. Luckily we have `JsonHelper` to help with this, and a config system with `ConfigContainer` which streamlines the process.

In addition to Codec there is also MapCodec. This is a specialised codec for encoding and decoding maps with string keys, whereas normal Codecs are for direct values. A MapCodec generally consists of a bunch of string keys and the codec to use for each of these keys. It parses all the sub-values and creates an object containing them. You can easily convert a normal Codec to a MapCodec do by running `Codec::fieldOf` and giving it a name. Another option is to use `Codec::optionalFieldOf` which gives you an optional version of the MapCodec, where it will return empty if the value is omitted. This is useful because null values generally don't work well with codecs. You can also just use this function to give the codec a default value which will be used instead. A MapCodec can easily be converted into a normal Codec when necessary by calling `MapCodec::codec`.

Defining a codec for a class you made is easy. If it's a singleton, you can just use the `MapCodec::unitCodec` or `MapCodec::unit` functions.
For enums, you'll want to use `StringRepresentable`. Basically, have the enum implement `StringRepresentable` to provide each value with a name, and then use `StringRepresentable.fromEnum(<enum>::values)`.
For other objects, you'll want to use `RecordCodecBuilder::create` or `RecordCodecBuilder::mapCodec`. It will look something like this:
```
RecordCodecBuilder.create(
	instance -> instance.group(
		<a codec>.fieldOf(<some name>).forGetter(<function to fetch this value from the object>),
		<a map codec>.forGetter(<function to fetch value from object>),
		<as many objects as you wish (or well, there is an upper limit, split into smaller map codecs to avoid this)> ---||---
	).apply(instance, <object factory>)
)
```
The object factory takes all types you'd get from the codecs provided in the group function and expects you to construct the object you're fetching the data from.
Note how you can specify MapCodec directly without using fieldOf. When doing so, all string fields from that MapCodec will be added to the MapCodec you're creating at the bottom level. You can still use fieldOf for MapCodecs if you still want to put the data in a sub-element. Using fieldOf is mandatory for normal non-map codecs.
Using the `create` function creates a normal Codec. You can always use `mapCodec` instead to get the MapCodec (`create` literaly just runs `mapCodec` and then runs `MapCodec::codec` on the result).

You can also `Codec::xmap` an existing codec directly to convert it to another type, or `Codec::comapFlatMap` if you know that some values won't map to valid types (you then return a DataResult instead of the object directly, allowing you to specify an error message). There are some additional ones like `Codec::flatXmap` and `Codec::flatComapMap`, but they're more neash (although I've used flatXmap a bit since I didn't know comapFlatMap existed at the time).

You can easily get a codec for a list with `Codec::listOf` and a map with `Codec::unboundedMap`.
There's also `Codec::lazyInitialized` for those edge cases where you get an exception because it's classloading stuff too early (or for simpler recursive codecs).
There are also some more fancy ones like `Codec::recursive` for recursive codecs and `Codec::dispatchedMap` for cases where each key should have a separate codec.
Any codec can also be "dispatched" with the `Codec::dispatch` or `MapCodec::dispatch`. This turns the codec into a polymorphic resolver where the type parameter from the codec you ran dispatch on is used to find which codec is used for the value. This is often used for dynamic registries and is how codecs like Recipe, Dialog and many of the enchantment and worldgen codecs work behind the scenes.

It is of course possible to extend Codec directly, but this should only be used in edge cases, such as wanting to access RegistryOps directly in a non-trivial way.


However, sometimes when using codecs you might find that for some reason it just won't encode or decode your data, complaining that some registry can't be found.
For example, if you have an item with enchantments on it, it might complain that the enchantment registry is not accessible. This is because you need to use RegistryOps.
RegistryOps is a wrapper around the ops you have which gives certain codecs (specifically `RegistryFixedCodec` and `ContextRetrievalCodec`) access to registries, as they won't work otherwise. It's also necessary for some codecs such as `HolderSetCodec` and `RegistryFileCodec` to be able to use references (if you don't use RegistryOps it will always encode them inline and tags won't work). You can create it with `RegistryOps::create`, or `HolderLookup.Provider::createSerializationContext`.
You'll need your registry access to use these functions, which is usually either passed as an argument, or it can be fetched from your nearest level, entity or server with `registryAccess` (in some cases you might also want `reloadableRegistries` from the server to access things like the predicate registry).
Don't worry about RegistryOps when working with nbt. Your ReadView/WriteView gets a registry manager behind the scenes, so as long as you use the read and put functions you'll be fine.
Most of Mojang's dispatched sub-codecs will allow RegistryOps, so you usually don't have to worry about adding a MapCodec with RegistryOps to a registry.
But there is one notable exception, TimerQueue. For situations like these, we have `ObjectStorage`, which stores the object as raw data and takes a HolderLookup.Provider whenever you want to fetch the data.

Some extra tips:
Don't use `MapCodec::assumeMapUnsafe`. If you need a Codec in the form of a MapCodec, use `Codec::fieldOf` instead and give it an appropriate name, or make sure your codec is stored as a MapCodec directly.
`MapCodec::orElse` is an alternative to `Codec::optionalFieldOf`. The difference is that `Codec::optionalFieldOf` never serializes the default value whereas `MapCodec::orElse` always does.
`Codec::lenientOptionalFieldOf` is an alternative to `Codec::optionalFieldOf`. `Codec::optionalFieldOf` will only fall back to the default value (or empty) if the value is completely missing, whereas `Codec::lenientOptionalFieldOf` falls back if the value is specified but invalid as well.
Take care to not use `Codec::orElse` instead of `MapCodec::orElse`. Using that in a MapCodec will still error if the value is missing. Basically, always use `fieldOf` first, then `orElse`. Although in my opinion it's better to just use `optionalFieldOf`.

You can use `RegistryOps::retrieveGetter` to fetch a registry directly if you need that in your object.
You can also use `RegistryOps::retrieveElement` to fetch a specific hardcoded registry entry from the registries, but there's usually not a good reason to do this.
Note that both of these depend on RegistryOps.
You can also use `ExtraCodecs::retrieveContext` if you wish to access the DynamicOps directly.



# Mixins:
Mixins are your best friends. They let you modify Mojang's code to do whatever you want. But with great power comes great responsibility.
Each mixin class must be in the designated mixin package and be referenced in the mixin config.
Mixin classes can contain other nested mixin classes, but they cannot contain nested non-mixin classes. Normal classes can however contain nested mixin classes (provided they are in the mixin package).
Note that a mixin doesn't allow public static methods unless the mixin is an accessor, so you'll want to make any static methods you define inside them private (or just move them to a helper class).
Also, you'll generally want to avoid @Overwrite, @Redirect and @ModifyConstant since they break other mods (hence why they're not documented here, and you basically never need them anyway, at least when MixinExtras is available, which is shipped by default in Fabric, Quilt and NeoForge these days).

**@Mixin**: Specifies the class the mixin should target. You can also specify the class as a string in the targets parameter if for example it is private.
You also use this parameter to choose the priority of your mixin, where mixins with a lower priority get applied first.
Now, getting applied first does not necessarily mean your code runs first. If you use the HEAD injection point, then your code will run first if your mixin applies last, for example.
You can also specify if you want your mixin to be remapped or not (in the future you will generally want this to be set to false, unless you're working with 1.21.11 and lower, in which case you want it set to true for Minecraft classes).
You can also toggle this individually on each injector as needed.  
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/Mixin.html

**@At**: Used to select where your code gets injected.
If an injection point has an ordinal parameter, then that can be used to choose which match should be selected if multiple are available.
If ordinal is not specified, it usually defaults to selecting all of them.  
**Injection Points**:  
**HEAD**: Targets the beginning of the method.  
**RETURN**: Targets just before all return statements (only one of them will run at any given time).  
**TAIL**: Targets just before exiting the method at the far bottom (i.e. like RETURN, but skips return statements in the middle).  
**INVOKE**: Injects just before a method call.  
**FIELD**: Injects just before accessing or modifying the given field.  
**CONSTANT**: Injects before the given constant. You specify the constant's value in the arguments string.  
**NEW**: Matches an object creation with a constructor, i.e. a new statement in the code. You can usually target this point with invoke by specifying the `<init>` function as well. With that said, unlike `<init>`, this injector actually returns the constructed object, making it useful for @ModifyExpressionValue.  
**INVOKE_STRING**: Targets methods with a single literal string as the argument and returning void. Primarily intended to help target the profiler::startSection method calls.  
**INVOKE_ASSIGN**: Injects immediately *after* a method invocation using a similar selection system as INVOKE, but only if the method has a return value. Note that it often still runs before the return value is assigned to whatever variable it is assigned to, so use @ModifyExpressionValue instead if you want to change or capture the value.  
**JUMP**: Used to target specific bytecode instructions. This is hard to use and should generally only be used as a last resort.  
**MIXINEXTRAS:EXPRESSION**: A more modern way to target your injection points. Especially useful when you want to modify the condition of an if (or while) statement. More info: https://github.com/LlamaLad7/MixinExtras/wiki/Expressions  
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/InjectionPoint.html

The target of at parameters needs to be written in valid MemberInfo notation.
For example, to target the method `void printStrAndInt(String str, int i);` in the class `net.example.Helper` you would type: `Lnet/example/Helper;printStrAndInt(Ljava/lang/String;I)V`.
To target the field `long[] f;` in the nested class `net.example.Helper.Sub` you would type `Lnet/example/Helper$Sub;f:[J`.
Please see Oracle's documentation for which letter matches which primitive type: https://docs.oracle.com/javase/specs/jvms/se14/html/jvms-4.html#jvms-4.3.2
Inside the top-level annotations such as @Inject, it is often enough to specify the method name. As of 26.0, it should also be sufficient in @At annotations as well, but if you're working with 1.21.11 or older you'll need to use fully qualified names in the @At targets for remapping to work. If the target method has multiple overloads you may also wish to partially qualify the statement with a parameter list to select a specific overload.
It is highly recommended to use an IDE with an extension for Mixin support (such as IntelliJ with Minecraft Development) to help autocomplete these.
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/struct/MemberInfo.html

**@Inject**: Basically creates an event handler at the given position in the code, allowing you to add additional code to that location.
You also get a CallbackInfo object which you can use to cancel the method prematurely.
If the method you're injecting into has a return value, you get a CallbackInfoReturnable instead, which lets you set the return value (which also cancels the event).
Note that even for methods with a return value, the return type of the mixin method is always void.
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/Inject.html
Warning: Cancelling a mixin will stop other mixins in the same method from running, only do so if you want to stop the rest of the method from running!
If you just want to modify the return value, use @ModifyReturnValue instead.
It also has a feature for capturing local variables, but you probably want to use @Local instead in most cases.

**@ModifyVariable**: Lets you modify a variable in the code. It will select a variable matching the type of the first input and output parameter of the function. If you want to modify multiple variables at once, you might want to use @Inject with @Local instead. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/ModifyVariable.html

**@ModifyArg**: Lets you select a method call and replace one of the arguments sent to it. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/ModifyArg.html

**@ModifyArgs**: Like @ModifyArg, but you can replace multiple variables at once. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/ModifyArgs.html

**@ModifyExpressionValue**: Allows you to modify basically any value you can think of. It essentially wraps around whatever you target with it, allowing you to replace the return value of a method, or the value fetched from a field without touching the field itself. This is also the best way to modify literals (hardcoded values). It works in a similar way to @ModifyVariable, giving you the current value and expecting you to return the value you want. More info: https://github.com/LlamaLad7/MixinExtras/wiki/ModifyExpressionValue

**@ModifyReturnValue**: Lets you change what value is returned from the given method. Like ModifyVariable, you get the current return value and are expected to return the return value you actually want to return. More info: https://github.com/LlamaLad7/MixinExtras/wiki/ModifyReturnValue

**@WrapOperation**: Intercepts a method call, allowing you to call your own function instead to replace the return value. The last parameter is the original function you are wrapping, make sure to call it properly when you want to preserve vanilla/other mod behaviour! Usually, you'll want to use @ModifyExpressionValue instead to avoid mistakes, but this is useful if you want to completely prevent the original method from running at all (for example to boost performance). If you ever feel like using @Redirect, it's usually better to use this instead (unless you want the game to crash when someone else is also trying to use @WrapOperation on the same method call, then use @Redirect). More info: https://github.com/LlamaLad7/MixinExtras/wiki/WrapOperation
Note, to target instanceof checks, you want to use the constant parameter in the annotation to specify the type you are looking for.

**@Constant**: Basically a direct interface for the CONSTANT injection point, which is used in @ModifyConstant and @WrapOperation. You probably won't use it very often. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/Constant.html

**@WrapWithCondition**: Special case of @WrapOperation when there is no return value. You just return true if the code should go ahead as usual, and false to skip it. There are 2 versions of this one, so make sure to pick the non-deprecated one! More info: https://github.com/LlamaLad7/MixinExtras/wiki/WrapWithCondition

**@Local**: Lets you capture an individual local variable (or the input parameters) of a method. Basically, you add the variable type and a variable name of your own choosing and add this annotation on the left of the type. You can also use a LocalRef (or one of the primitive local ref types for primitive variables, i.e. LocalIntRef) if you wish to be able to modify the variable, but that is optional. More info: https://github.com/LlamaLad7/MixinExtras/wiki/Local

**@Share**: Lets you inject your own variables to share between mixins in the same method. Works in a similar way to @Local. More info: https://github.com/LlamaLad7/MixinExtras/wiki/Share

**@Coerce**: Can be placed on parameters to allow them to accept values that don't exactly match the input type. Useful when dealing with private classes, but can also be used to merge mixin handlers.
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/Coerce.html

**@WrapMethod**: Lets you effectively replace a specific method. If you ever felt like using @Overwrite, you should use this instead. More info: https://github.com/LlamaLad7/MixinExtras/wiki/WrapMethod

**@ModifyReceiver**: Lets you replace the object calling a method, but without touching variable it's stored inside. More info: https://github.com/LlamaLad7/MixinExtras/wiki/ModifyReceiver

**@Unique**: Used when you want to add methods or fields to the class for your own use. For example you want to save additional data to the player, or want a simple helper function that doesn't make sense to move to a separate helper class. Basically, when you add this annotation to your field/method, the mixin library will make sure it doesn't conflict with other mods. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/Unique.html

**@Shadow**: Indicates that the method or field exists in the target class, allowing you to access them.
When targeting a static method, just have an empty implementation or one that throws an exception, that code will never run.
For non-static methods, you can mark the mixin class as abstract so you don't need to define them. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/Shadow.html
Your IDE might also ask you to add some additional annotations such as @Final (since declaring a field as final would cause issues).

**@Mutable**: Can be applied on @Shadow fields that are final in the base class to allow modifying them.

**@Pseudo**: Prevents the game from crashing and removes all warnings printed when the mixin target class does not exist. Useful when creating mixins into other mods. More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/Pseudo.html

**@Slice**: Used to limit the scope of the mixin, making only targets between two specific @At parameters selectable.
This is usually specified in the slice parameter of the injector method.
Sub-arguments (such as @At) usually only have a string field for slice, but the top level injector usually allows specifying an array of slices and giving each one a unique name which can be specified in sub-parameters.
Useful for large and messy functions where it's difficult to get the correct value for ordinal, or if you need to use a harder to use injection point such as JUMP.  
More info: https://javadoc.io/doc/net.fabricmc/sponge-mixin/latest/org/spongepowered/asm/mixin/injection/Slice.html

## A few extra tricks:
It is highly recommended to install the IntelliJ Minecraft development plugin (or equivalent on whatever IDE you want to use), as that lets your IDE help you type fully qualified method qualifiers which are a pain to type manually.  
Sometimes it can be difficult to find the actual point your mixin should target. In these cases, it might help to look at the compiled bytecode, which you can access with View -> Show Bytecode in IntelliJ.   
If you need to access members of a parent classes of the class you're mixing into, you can simply extend that class with your mixin. Your IDE will probably ask you to implement the constructor, which you can do safely as it will be ignored.  
Feel free to mark your mixin class as abstract to avoid needing to define additional methods.  
If you're having trouble debugging your mixins, try exporting them: https://wiki.fabricmc.net/tutorial:mixin_export  
More tips: https://wiki.fabricmc.net/tutorial:mixin_tips  
If you wish to have a variable be shared between multiple methods, the best solution is to add a field to the class. If you wish to share a variable between multiple mixins, you'll probably need to use a global variable. To keep this "global" variable hidden, you can put it in a class in the mixin package and mark it as protected, or even make all mixins that use it nested classes inside a class containing the variable and make the variable private. In this case I recommend using ThreadLocal, just in case the method is ever called from other threads. If you want a variable to be shared between multiple mixins in the same method, please use @Share instead.  
There's nothing wrong with having multiple mixins target the same class. Usually, people put all code related to a specific class in the same mixin, but you can structure your project in a different way, such as separating mixins by feature. You can also put mixins in sub-packages inside the mixin directory, you just can't move them outside the mixin package.  


## Accessor Mixins:
Special mixins that are always interfaces even if the target class is a class. Used to let you access private fields and functions.  
Public static methods are allowed here.  
By default, they will target variables/methods with a name matching the name of the mixin method.  
For example, a getter for the variable/method `var` would be `getVar`, a setter would be `setVar` and invoker would be `callVar`, but you can specify the variable/method name manually in the annotation parameters.  
@Accessor: Used to make getters and setters to private fields.  
Getter: Takes zero arguments and returns the type of the variable you want to access.  
Setter: Takes one argument (the input argument name does not matter) with the type of the variable you want to access and returns void.  
Note that if a field is final, you need to add @Mutable as well to the setter (but not the getter).  
@Invoker: Takes the same arguments and has the same return type as the method you want to access (the input argument names once again does not matter).  
See @Shadow above for how to deal with static methods (public static methods are allowed in accessors).  


## Interface Injection:
You can inject additional interfaces into classes by simply implementing them in your Mixin class. You can then safely use @Override to implement the methods.  
When you've done this, you can safely cast objects of that type to your interface to access these methods.  
Additionally, you can add them to your fabric.mod.json which will make your dev environment update so you can see the methods without typecasting (when doing this I recommend giving the methods in the interface an empty default implementation as otherwise your IDE might think they're not implemented when extending the base class). More info: https://wiki.fabricmc.net/tutorial:interface_injection  
To specify generics in fabric.mod.json interface injection, add <> at the end and insert your type qualifier. For example `<Ljava/lang/String;>` places the string type and `<TF;>` places the type variable F, and `<TT;TU;Ljava/lang/String;>` inserts type variables T and U, and then the explicit String class type in that order.  
Note that for best mod compatibility you should add the mod id or mod namespace to the function name so it won't conflict with other mods.  
The general standard is to use `modid$name` (or at least that's what the IntelliJ extension recommends).  


# Access Wideners:
Useful when you want to access private classes, extend final classes/records or override final methods.  
More info: https://wiki.fabricmc.net/tutorial:accesswideners  
Note that transitive access wideners don't work as expected within multi-modular projects (at least they didn't when I wrote this, if it's been fixed you should remove this line). This means you'll have to define the access widener you need in every module where you need it, while accessors only need to be defined once in a common dependency.  


# Extending enums with Fabric ASM:
Sometimes, you want to add an additional enum constant since a function you want to run only accepts an enum for that argument, even though additional values make sense for that argument.  
Some people have had success with mixins (https://github.com/SpongePowered/Mixin/issues/387#issuecomment-888408556), but using Fabric ASM is probably safer.  
First, add a new entry point with the name `mm:early_risers` implementing `Runnable` (or reuse one that already exists). This will run very early, so avoid accessing classes you don't need.  
Then use `ClassTinkerers::enumBuilder` within the run function to specify the class path and name of the enum you want to extend, and the types of the arguments to its constructor.  
There is one function which takes the classes directly, and one which takes the internal string names instead if you can't access the classes directly.  
Then, you can add all enums you desire with `EnumAdder::addEnum`, which takes the new enum value's string name and any arguments to pass to the constructor.  
There is one version of this function which simply accepts the arguments directly, and one which takes a supplier which is useful if any of the classes used for the inputs can't be accessed at this stage.  
There is also `EnumAdder::addEnumSubclass` which is used in the case where the enum is abstract and you need to implement methods, but using it is a bit complicated and involves adding a mixin which is not registered in the mixin config.  
When you're ready, you can invoke `EnumAdder::build`.  
Then, you can access your new enum value at runtime with `ClassTinkerers::getEnum` and passing the enum class and the name you provided in the addEnum method. Note that this method uses a for-loop so you should ideally fetch it once and store it in a constant somewhere. You could also use `Enum::valueOf` (which caches the values), but I can't help but wonder why they didn't do that themselves. It can't be as simple as that they didn't realise that function existed, surely? Maybe they were worried about the valueOf cache being instantiated too early, potentially breaking other mods?  
More info: https://github.com/Chocohead/Fabric-ASM  
Fabric ASM also has access transformers, which serve a similar purpose to access wideners. There is generally no reason to use them however. I'm pretty sure the only reason they exist in Fabric ASM is that they were added before access wideners were implemented in Fabric, and there simply wasn't a good reason to remove them.  
Fabric ASM also has more advanced methods for modifying classes, but you should only touch that if you're doing some crazy stuff and are certain it's the only good way forward.  

Note, when working with older versions (1.21.11 and lower), you generally want to use the *intermediary* name rather than the named name, and then remap it to the correct runtime name using `FabricLoader::getMappingResolver`. You can find the intermediary name of a class in Fabric Yarn: https://github.com/FabricMC/yarn


# Scheduling Events:
The "proper" vanilla way of scheduling events is to use `net.minecraft.world.level.timers.TimerQueue`, which is what the game uses for the /schedule command. The main benefit of this one is that scheduled events stay scheduled between reboots. The main downside is that each event type must be registered (so each event type needs a class and a codec, you can't just "give it a lambda").
You can fetch it with: `server.getWorldData().overworldData().getScheduledEvents()`.
To use this scheduler, each event type you want to schedule must be registered with a MapCodec to `net.minecraft.world.level.timers.TimerCallbacks`.

Careful here, the game does not use RegistryOps when encoding and decoding the scheduler, which means some codecs won't work because they can't access the required registries. More specifically, you'll need to avoid `RegistryFixedCodec`, `RegistryOps::retrieveGetter` and `RegstryOps::retrieveElement` as well as any codec which depends on them. I would also be wary of HolderSetCodec, because if RegistryOps is unavailable it will fall back on its entry codec, which is very often a RegistryFixedCodec (but does not have to be). RegistryFileCodec is generally safe, but always double check the element codec since that's what will be used when RegsistryOps is missing. `Registry::byNameCodec`, `Registry::holderByNameCodec` and `Registry::referenceHolderWithLifecycle` are all safe since they don't need RegistryOps (at least for now).

If you want to use an object which has a registry-dependent codec, you can use ObjectStorage. This gives you a wrapper object which simply contains the raw data, which you can then parse when needed. Just keep in mind that it does not do any caching of the data by itself, so avoid parsing the data more than once.

Sometimes you might just want to delay something until the end of this tick. Maybe you're inside a loop of the list you want to modify, or you want to let other entities tick on something before you act. Then you can call `MinecraftServer::scheduleWithResult`. It takes a simple lambda as an argument which is the code you want to run. Note that your lambda has one input parameter, a CompletableFuture. This CompletableFuture can be safely ignored, as it's only used as the return type of the scheduleWithResult function, making it easy to make other threads wait for a return value. You can mark it as completed if you want, but you don't have to. Just be sure to not call join on the return value when running this function on the server thread, as that would cause a deadlock.

You can also refer to `TaskScheduler` which uses these systems under the hood, making it easier to remember. We also include a custom `Throwaway` schedule event which is hardcoded to never be saved, allowing you to schedule unimportant things such as particles and sounds without needing to define a codec.


# DataFixer:
When coding mods that store items (or other types of data that Mojang likes to change every update), you probably want to hook into Mojang's DataFixer, otherwise people might lose their items. This can be done by adding Mixins into relevant schemas. To find which schemas you need to add a mixin into you can look at the TypeReference relevant for the situation and press the Find Usages button. Note that updates sometimes overwrite existing types in a new schema, so you may need to inject similar mixins into multiple schemas.
A few pointers:
- Don't try to make schemas recursive if they are marked as non-recursive, this will likely introduce other issues down the road (speaking from experience).
	- So, if you store player data inside of player data, you may want to skip adding it to the DataFixer and just call on the datafixer manually when loading instead.
		- Player data always saves the DataVersion, so no extra work is necessary in this case (you might need to write this parameter manually for other data types).
		- You can use `PlayerDataHelper::updatePlayerData` for this purpose.
	- Feel free to make new types you introduce recursive though.
- If you want to add a datafixer for a SavedData, you'll need to put your TypeReference inside the DataFixTypes enum.
	- The easiest way to do this is to use Fabric ASM.
	- You can find some examples in metacraft-cutscenes and metacraft-saved-items.
- One way to avoid needing to define DataFixer extensions is to reuse vanilla data components.
	- For example, say you want to add a custom inventory container.
		- Instead of writing the items to data manually or defining a new component, you can just store the items inside the "minecraft:container" data component since that will be covered by the DataFixer already.
			- Just remember that item components are meant to be immutable. So make sure to create a new component every time the inventory is changed, don't just modify the list.
		- Also, remember that you can put data components on entities and block entities as well, not just items.
- You technically don't need to add any DataFixers until it's time to update to the next version, but then you might forget it and miss it in testing.
	- With that said, the new update might overwrite your schema modification in a new schema, forcing you to update the datafixer either way.


# Modular Development:

- Circular dependencies are not supported.
	- If you want to add compatibility between 2 modules, you have 2 options.
		1. Add all the compatibility-code to one of the modules and make it depend on the other one.
		2. Create a third module with all the compatibility code and have it depend on both.
	- Custom events might be helpful: https://wiki.fabricmc.net/tutorial:events?s%5B%5D=eventfactory

	
# Testing:
If you feel like writing unit tests in mods (100% optional), then you can use Fabric's [JUnit Integration](https://docs.fabricmc.net/develop/automatic-testing).
This is already available in METAmods, just add a test directory to the module you want to test something for and add a new class (see existing modules for examples). You use it the same way you'd normally use JUnit. Note that we currently don't use the Game Test framework.
When making a test class, you're probably going to want to run `TestHelper::init` in a @BeforeAll method at the beginning of your test.
This function takes ModInitializers for all mods you want to have active, as well as an opportunity to register additional registry objects if necessary.
If you have multiple test classes you might want to move this into a separate BeforeAllCallback class and add it via an @ExtendWith (see metacraft-lib for examples).
If you want to take advantage of [Minecraft's Testing Framework](https://minecraft.wiki/w/GameTest), you can launch the test server with `TestHelper::runTestServer`. If you want to run custom code in your tests, you want to register custom test functions in your `TestHelper::init` function. You can put any test-specific datapack resources (such as test instances or environments) directly in the test module. Please see metacraft-lib for examples.
