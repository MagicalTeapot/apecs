# apecs: A Petite Entity Component System
A header-only, very small entity component system with no external dependencies. Simply pop the header into your own project and off you go!

The API is very similar to EnTT, with the main difference being that all component types must be declared up front. This allows for an implementation that doesn't rely on type erasure, which in turn allows for more compile-time optimisations.

Components are stored contiguously in `apx::sparse_set` objects, which are essentially a pair of `std::vector`s, one sparse and one packed, which allows for fast iteration over components. When deleting components, these sets may reorder themselves to maintain tight packing; as such, sorting isn't currently possibly, but also shouldn't be desired.

This library also includes some very basic meta-programming functionality, found in the `apx::meta` namespace.

This project was just a fun little project to allow me to learn more about ECSs and how to implement one, as well as metaprogramming and C++20 features. If you are building your own project and need an ECS, I would recommend you build your own or use EnTT instead. This was originally
implemented using coroutines, however the allocations caused too much overhead, so I replaced them
with an iterator implemenetation, before realising that coroutines were the wrong tool to begin
with, and the correct tool was C++20 ranges. Ultimately the coroutine implementation was just
transforming and filtering vectors of entities and components, which is now expressed directly
in the code and no longer uses superfluous dynamic memory allocations.

## The Registry and Entities
In `apecs`, an entity, `apx::entity`, is simply a 64-bit unsigned integer. All components attached to this entity are stored and accessed via the `apx::registry` class. To start, you can default construct a registry, with all of the component types declated up front
```cpp
apx::registry<transform, mesh, light, physics, script> registry;
```
Creating an empty entity is simple
```cpp
apx::entity e = registry.create();
```
Adding a component is also easy
```cpp
transform t = { 0.0, 0.0, 0.0 }; // In this example, a transform consists of just a 3D coordinate
registry.add(e, t);

// Or more explicitly:
registry.add<transform>(e, t);
```
Move construction is also allowed, as well as directly constructing via emplace
```cpp
// Uses move constructor (not that there is any benefit with the simple trasnsform struct)
registry.add<transform>(e, {0.0, 0.0, 0.0});

// Only constructs one instance and does no copying/moving
registry.emplace<transform>(e, 0.0, 0.0, 0.0);
```
Removing is just as easy
```cpp
registry.remove<transform>(e);
registry.remove_all_components(e);
```
Components can be accessed by reference for modification, and entities may be queried to see if they contain the given component type
```cpp
if (registry.has<transform>(e)) {
  auto& t = registry.get<transform>(e);
  update_transform(t);
}
```
There are varidic versions of the above in the form of `*_all`
```cpp
if (registry.has_all<transform, mesh>(e)) {
  auto [t, m] = registry.get_all<transform, mesh>(e);
  update_transform(t);
}
```
If you want to know if an entity has *at least one* of a set of components:
```cpp
registry.has_any<box_collider, sphere_collider, capsule_collider>(e);
```
There is also a noexcept version of `get` called `get_if` which returns a pointer to the component, and `nullptr` if it does not exist
```cpp
if (auto* t = registry.get_if<transform>(e)) {
  update_transform(*t);
}
```

Deleting an entity is also straightforward
```cpp
registry.destroy(e);
```
You can also destroy any span of entities in a single call:
```cpp
registry.destroy({e1, e2, e3, e4});
```
Given that an `apx::entity` is just an identifier for an entity, it could be that an identifier
is referring to an entity that has been destroyed. The registry provides a function to check this
```cpp
registry.valid(e); // Returns true if the entity is still active and false otherwise.
```
The current size of the registry is the number of currently active entities
```cpp
std::size_t active_entities = registry.size();
```
Finally, a registry may also be cleared of all entities with
```cpp
registry.clear();
```

## Iteration
Iteration is implmented using C++20 ranges. There are two main ways of doing interation; iterating over all entities, and iterating over a *view*; a subset of the entities containing only a specific set of components.

Iterating over all
```cpp
for (auto entity : registry.all()) {
  ...
}
```
Iterating over a view
```cpp
for (auto entity : registry.view<transform, mesh>()) {
  ...
}
```
When iterating over all entities, the iteration is done over the internal entity sparse set. When iterating over a view, we iterate over the sparse set of the first specified component, which can result in a much faster loop. Because of this, if you know that one of the component types is rarer than the others, put that as the first component.

It is common that the current entity is not actually of direct interest, and is only used to fetch components. For this, there is `view_get` which instead returns a tuple of components instead of the entity id:
```cpp
for (auto [t, m] : registry.view_get<transform, mesh>()) {
  ...
}

// The above is clearer than the more verbose and error prone:
for (auto entity : registry.view<transform, mesh>()) {
  auto& t = registry.get<transform>(entity);
  auto& m = registry.get<mesh>(entity);
  ...
}
```
Note that you also don't need to care about constness in the `view_get` case. If the registry is not const, the values in the tuple will be references, and if you are accessing through a `const&` registry, the components will also be `const&`. If you dont have a `const&` to the registry, you can enforce it by creating one and viewing through that:
```cpp
const auto& cregistry = registry;
for (auto [t, m] : cregistry.view_get<transform, mesh>()) {
  // t and m are const& in this context
}
```

## Other Functionality
The registry also contains some other useful functions for common uses of views:

### Finding an Entity via a Predicate
You can look up an entity that satisfies a callback via
```cpp
registry.find([](auto entity) -> bool { ... });
```
By default this loops over all entities and returns the first one satisfying the given predicate,  returning `apx::null` if there is no such entity. This can be optimized by looping over a view if desired:
```cpp
registry.find<transform>([](auto entity) -> bool { ... });
```

### Copying Entities
It might desirable to duplicate entities within a registry. More generally, given
two different registries of the same templated type, it may also be useful to be
able to copy an entity from one registry to another. For this, use `apx::copy`:
```cpp
template <typename... Comps>
entity copy(entity entity, const registry<Comps...>& src, registry<Comps...>& dst);
```
The given entity must be a valid entity in the src registry.

### Deleting Entities via a Predicate
Deleting entities in a loop is undefined behaviour as you could be modifying the container you are iterating over. To delete a set of entities safely 
```cpp
registry.destroy_if([](auto entity) -> bool { ... });
```
This can also take template parameters to do the loop over a view as well.

## Metaprogramming
To implement many of these features, some metaprogramming techniques were required and are made available to users. First of all, `apx::tuple_contains` allows for checking at compile time if a given `std::tuple` type contains a specific type. This is used in the component getter/setter functions to give nicer compile errors if there is a type problem, but may be useful in other situations.
```cpp
static_assert(apx::tuple_contains_v<int, std::tuple<float, int, std::string>> == true);
static_assert(apx::tuple_contains_v<int, std::tuple<float, std::string>> == false);
```
When destroying an entity, we also need to loop over all types to delete the components and to make sure any `on_remove` callbacks are invoked. This can be done with `apx::for_each`
```cpp
apx::meta::for_each(tuple, [](auto&& element) {
  ...
});
```
This of course needs to be generic lambda as this gets invoked for each typle in the tuple.

In extension to the above, you may also find yourself needing to loop over all types within a reigstry. This can be achieved by creating a tuple of `apx::meta::tag<T>` types and extracting the type from those in a for loop. The library provides some helpers for this. In particular, each registry provides an `inline static constexpr` version of this tuple as `registry<Comps...>::tags`:
```cpp
apx::meta::for_each(registry.tags, []<typename T>(apx::meta::tag<T>) {
  ...
});
```

## Upcoming Features
- The ability to specify `const` in the getters, such as `registry.get<const transform, mesh>(entity)`.