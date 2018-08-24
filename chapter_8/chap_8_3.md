# 使用 @ngrx/entity 提升生产效率

ngrx 很酷，但是我们经常发现自己为不同类型的数据写几乎完全相同的 reducer 逻辑和选择器，这很容易出错，而且真的很烦。 ngrx 团队估计也收到了很多这种抱怨，于是他们开发了 `@ngrx/entity` 这个类库，帮我们简化开发。

在 ngrx 中，我们在 store 中存储不同类型的状态，这通常包括：

* 商业数据，例如任务、项目等
* 某些 UI 状态，例如 UI 的某些设置项、加载进度等

Entity 是什么呢？有面向对象编程经验的同学知道这是实体的意思，那么什么又是实体呢？一般来说，实体是一个可持久化的领域对象，在后端，实体通常表示关系数据库中的表，并且每个实体实例对应于该表中的行。在 Redux 中，我们可以把 Store 看成数据库，所以实体就是代表我们业务数据了，比如项目：

```ts
export interface Project {
  id: string | undefined;
  name: string;
  desc?: string;
  coverImg?: string;
  enabled?: boolean;
  taskFilterId?: string;
  taskLists?: string[];
  members?: string[];
}
```

一般来说，实体都有一个名为 id 的唯一标识符字段，可以是字符串或数字。我们存储在 Store 中的大多数数据都是实体，这些实体在 store 中以数组的形式存储是非常自然的，但这种方法可能有几个潜在的问题：

* 如果我们想根据已知 id 查找实体，我们将不得不遍历整个集合，这对于非常大的集合来说可能是低效的
* 如果使用数组，可能会不小心在数组中存储相同实体的不同版本（具有相同的 id ）

由于 store 充当内存客户端数据库，所以将业务实体存储在它们自己的内存数据库“表”中是有意义的，并为它们提供类似于主键的唯一标识符。然后可以将数据扁平化，并使用实体唯一标识符链接在一起，较好的建模方法是将实体集合存储在字典中，实体的键是唯一 ID ，值是整个对象。

```js
{
    projects: {
        0: {
            id: 0,
            name: '测试项目1',
            desc: '这是一个测试项目1'
        },
        1: {
            id: 1,
            name: '测试项目2',
            desc: '这是一个测试项目2'
        },
        ...
    }
}
```

这种存储方式使得通过 id 查找实体非常简单，例如，为了查找 id 为 1 的项目，我们只需写成，注意不要和数组弄混了，这是取对象的 key 为 1 的值：

```ts
projects[1]
```

但是这种结构带来一个排序的问题，因为字典是没有顺序的，那么我们是否可以结合两种数据结构的优点呢？有的，那就是...我们既使用字典，又使用数组就好了。哈哈，但是为了减少数据的重复，我们的数组是一个只有 id 的数组：

```ts
{
    projects: {
        ids: [0, 1, ...]
        entities: {
            0: {
                id: 0,
                name: '测试项目1',
                desc: '这是一个测试项目1'
            },
            1: {
                id: 1,
                name: '测试项目2',
                desc: '这是一个测试项目2'
            },
            ...
        }
    }
}
```

这种类型我们可以定义成下面的样子

```ts
export interface ProjectState {
    ids: number[];
    entities: {[key:number]: Project};
}

```

当然这只是项目的状态，整个应用的状态可以这样定义

```ts
export interface State {
    projects: ProjectState:
    tasks: TaskState;
    ...
}
```

在这样的结构下面，大部分实体的 reducer 是非常类似的，大致看起来是下面的样子，只是实体不同而已：

```ts
const initialProjectState: ProjectState = {
    ids: [],
    entities: {}
}

export function sortByName(a: Project, b: Project): number {
  return a.name.localeCompare(b.name);
}

export function reducer(
    state = initialProjectState,
    action: ProjectActions): ProjectState {
    switch (action.type) {
        case ProjectActionTypes.AddProjectSuccess: {
            const project = action.payload;
            if(state.entities[project.id]) return state;
            const ids = [...state.ids, project.id];
            const entities = {...state.entities, {[project.id]: project}};
            return {...state, {ids: ids, entities: entities}};
        }
        default:
            return state;
    }
}
```

这就引出了我们开头说的，在大项目中写这种重复度较高的代码是很烦的，所以 `@ngrx/entity` 就来解救我们了
