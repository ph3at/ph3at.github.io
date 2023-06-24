---
title: C++ Static Reflection — Part 2
date: 2023-11-11
categories: [Programming]
tags: [programming,c++,reflection,ecs,ikaros]
author: alex
img_path: /assets/2023-11-11-CPP-Static-Reflection-2/
---

In [part 1](../CPP-Static-Reflection-1) of this series we investigated how [refl-cpp](https://github.com/veselink1/refl-cpp) can be used to enable some form of static reflection in modern C++.
Along the way, we established a running example of integrating this reflection mechanism into a tiny game engine prototype.

Now, in part 2, we will extend this integration even further.
Specifically, we will introduce the **component registry** and cover **attributes**.

## Where We Left Off

While putting together the first version of the EcsEditor, we discovered that [EnTT](https://github.com/skypjack/entt) (the entity component framework of the engine) does not allow us to simply iterate over all components associated with an entity.
Instead we have to iterate over all component types and check whether an entity possesses an instance of it.

```c++
ImGui::BeginChild("Entity", {windowSize.x * 0.7f, windowSize.y});
if (m_selectedEntity) {
    drawComponentEditor<Transform>("Transform", scene, *m_selectedEntity);
    drawComponentEditor<ModelComponent>("Model", scene, *m_selectedEntity);
    drawComponentEditor<SpinnerComponent>("Spinner", scene, *m_selectedEntity);
}
ImGui::EndChild();
```

Listing all components this way is undesirable as other subsystems utilizing our reflection mechanism would have to do the same, effectively violating the [DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

## The Component Registry

We therefore establish a dedicated place, where all component types are registered.
This is referred to as the **component registry**, not to be confused with EnTT's registry (seen previously as part of the `Scene`).

Let us first establish what data we want to store for each component type.

```c++
struct ComponentInfo {
    etl::string<64> name;
    int sortOrder = 0;
    entt::id_type id;
    bool hideInEditor = false;

    std::function<void(entt::handle)> addTo;
    std::function<void(entt::handle)> removeFrom;
    std::function<bool(entt::const_handle)> isPresentIn;
    std::function<void(editor::EditWidgetDrawer&, entt::handle)> drawEditWidget;
};
```

For each component type we store its name, id (using `entt::type_hash`), and some other metadata.
We also store operators for adding / removing the component to / from an entity, checking whether an entity has a component of this type, and drawing an edit widget.

`entt::handle` combines an `entt::entity` (which is effectively just an id) with the corresponding `entt::registry`.
Components can be managed through this handle with ease.

The `ComponentRegistry` itself is rather simple.
It stores instances of `ComponentInfo` for us to easily retrieve and use them.

```c++
class ComponentRegistry {
  public:
    ComponentRegistry();
    ComponentRegistry(const ComponentRegistry&) = delete;
    ComponentRegistry& operator=(const ComponentRegistry&) = delete;
    ComponentRegistry(ComponentRegistry&&) noexcept = delete;
    ComponentRegistry& operator=(ComponentRegistry&&) noexcept = delete;

    template <typename Component>
    void registerComponent(std::string_view name, int sortOrder = 0)
    {
        ComponentInfo info{
            .name = {name.data(), name.size()},
            .sortOrder = sortOrder,
            .id = entt::type_hash<Component>(),

            .addTo = [](entt::handle entity) { entity.emplace_or_replace<Component>(); },
            .removeFrom = [](entt::handle entity) { entity.remove<Component>(); },
            .isPresentIn = [](entt::const_handle entity) { return entity.try_get<Component>(); },
            .drawEditWidget = [](editor::EditWidgetDrawer& draw, entt::handle entity) {
                if (auto* component = entity.try_get<Component>()) {
                    draw(*component);
                }
            },
        };

        if constexpr (refl::is_reflectable<Component>()) {
            info.hideInEditor = has_attribute<editor::attr::Hidden>(refl::reflect<Component>());
        }

        addComponentInfo(info);
    }

    const auto& components() const { return m_sortedInfos; }
    const ComponentInfo* componentByID(entt::id_type) const;
    const ComponentInfo* componentByName(std::string_view) const;

  private:
    void addComponentInfo(const ComponentInfo&);

    static constexpr size_t MaxComponents = 32;

    etl::vector<ComponentInfo, MaxComponents> m_infos;
    etl::vector<const ComponentInfo*, MaxComponents> m_sortedInfos;
    etl::unordered_map<entt::id_type, const ComponentInfo*, MaxComponents> m_lookupByID;
    etl::unordered_map<etl::string_view, const ComponentInfo*, MaxComponents> m_lookupByName;
};
```
{: file="ikaros_component_registry.hpp"}

Upon registering a component, we fill in the fields for the corresponding `ComponentInfo` and store it.
In the code above we see a new thing we haven't looked at yet: `has_attribute`.
But more about this in a moment.

> Since `etl::vector` is a fixed-sized array which doesn't use heap allocation, pointers / references to elements won't be invalidated upon adding elements.
{: .prompt-info }

Registering a component is straightforward, we just have to call the `registerComponent` member function during engine initialization.
We commonly do this in the constructor of the corresponding system.
For instance, the `SpinnerSystem` registers the `SpinnerComponent` upon construction.

```c++
SpinnerSystem::SpinnerSystem(ComponentRegistry& cr)
{
    cr.registerComponent<SpinnerComponent>("Spinner");
}
```

Given what we can already achieve during compile-time using modern C++ and libraries like refl-cpp, we could probably implement the component registry in a `constexpr` way, where all components are registered during compile time.
Even further, we might be able to iterate over them the same way we can iterate over refl-cpp `FieldDescriptor`s, effectively eliminating any runtime overhead.
However, there is no practical benefit to this at the moment, and the code would likely be more complex.

### Where We Left Off, Again

With the component registry established, the undesired code piece can now be replaced.

```c++
class EcsEditor {
  public:
    // ...

    void tick(Scene& scene)
    {
        // ...
        ImGui::BeginChild("Entity", {windowSize.x * 0.7f, windowSize.y});
        if (m_selectedEntity) {
            drawComponentEditor(scene.entityHandle(*m_selectedEntity));
        }
        ImGui::EndChild();
        // ...
    }

  private:
    void drawComponentEditor(entt::entity_handle entity) const
    {
        EditWidgetDrawer drawer;

        for (const auto* component : componentRegistry.components()) {
            if (component->hideInEditor || !component->isPresentIn(entity)) {
                continue;
            }

            if (ImGui::TreeNode(component->name.c_str())) {
                if (component->drawEditWidget) {
                    component->drawEditWidget(drawer, entity);
                } else {
                    ImGui::TextDisabled("No editWidget defined");
                }
                ImGui::TreePop();
            }
        }
    }

    ComponentRegistry& m_componentRegistry;

    std::optional<entt::entity> m_selectedEntity;
```

Using `componentRegistry.components()`, we can now iterate over all (registered) component types and check whether the given entity possesses such a component.
If so, we invoke the `drawEditWidget` operator with the `EditWidgetDrawer` instance.

No more explicitly listing all components in various places!

## Attributes

Attributes offer a way of attaching additional information to a `FieldDescriptor`.

```c++
namespace ikaros::editor::attr {

// Prevents the type, field, or property to show up in the editor.
struct Hidden : refl::attr::usage::type,
                refl::attr::usage::field,
                refl::attr::usage::function {};

// Uses a slider widget instead of the regular drag widget.
template <typename T>
struct Slider : refl::attr::usage::field,
                refl::attr::usage::function {
    constexpr Slider(T min, T max) : min(min), max(max) {}
    T min;
    T max;
};

} // namespace ikaros::editor::attr
```

An attribute is just a type that may or may not contain some data.
By inheriting from types located in the `refl::attr::usage` namespace we define what it can be attached to.

For instance, drawing `exposure` and `gamma` as sliders while hiding `effectIndex`.

![Attributes Example](attributes.png)

```c++
REFL_TYPE(ikaros::PostProcessParams)
REFL_FIELD(exposure, ikaros::editor::attr::Slider(0.0f, 3.0f))
REFL_FIELD(gamma, ikaros::editor::attr::Slider(0.0f, 5.0f))
REFL_FIELD(effectIndex, ikaros::editor::attr::Hidden())
REFL_END
```

Using relf-cpp's `has_attribute` and `get_attribute`, we can check whether the attribute is attached and retrieve it in order to access the attached data.
However, we need refl-cpp's descriptor for this.
Here's the corresponding code in the `EditWidgetDrawer`:

```c++
class EditWidgetDrawer {
  public:
    bool field(const char* name, bool& value) { return ImGui::Checkbox(name, &value); }

    template <typename ReflDescriptor>
    bool field(ReflDescriptor member, const char* name, float& value)
    {
        if constexpr (has_attribute<attr::Slider<float>>(member)) {
            auto attr = get_attribute<attr::Slider<float>>(member);
            return ImGui::SliderFloat(name, &value, attr.min, attr.max);
        } else {
            return field(name, value);
        }
    }

    // ...

    template <typename T>
    bool operator()(T& object)
    {
        bool changed = false;

        if constexpr (refl::is_reflectable<T>()) {
            // Only consider members without the Hidden attribute.
            auto members = filter(refl::reflect<T>().members, [](auto member) { //
              return !has_attribute<attr::Hidden>(member);
            });

            auto fields = filter(members, [](auto member) { return is_field(member); });
            for_each(fields, [&](auto member) {
                changed |= field(member, member.name.c_str(), member(object));
                //               ↑
                //    Passing the descriptor along to the field member function.
            });
        }

        return changed;
    }

  private:
    // Fallback to silently accept all types that are not drawable.
    template <typename T>
    bool field(const char*, T&) { return false; }

    // Fallback for fields that do not take advantage of reflection attributes.
    template <typename ReflDescriptor, typename T>
    bool field(ReflDescriptor, const char* label, T& object)
    {
        return field(label, object);
    }
}
```

## What's Next?

In this part we've extended our infrastructure by introducing the `ComponentRegistry`.
Thanks to this element, we now have a dedicated utility for managing meta information on components.
Iterating over all components attached to a given entity is still its primary purpose.

We then looked into **attributes**, by which we can attach meta information to a type or to a specific member of a type.
Through this mechanism, semantic information is injected into the system, which allows for finer control in components that utilize the reflection mechanism.
For instance, drawing a slider widget with meaningful lower and upper bounds compared to just a plain numeric input field.

Next, we augment the `EditWidgetDrawer` to be more robust in what objects are accepted / rejected.
Furthermore, we add the ability to customize how certain objects (commonly components) are drawn.
We may also look into how enums can be supported in a user-friendly manner, so look forward to part 3!
