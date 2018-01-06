# Rescope Stores

## Events

Rescope use the concept of "Stability", so there only 4 events :

 - "stable"     (when the store current state is sync with the store data)
 - "unstable"   (see "Stability" below)
 - "update"     (when a store propag his data)
 - _"stableTree" (context only : when all child contexts are stable)_

## Stability

When state updates occurs, a store state stop being coherent with its results data.<br>
In Rescope stores, the role of the "apply" function is to make the store data predictable basing on its state.

This is done in 2 way :
 - In a synchronized way, by returning a new data hashmap from the "apply" fn

```jsx

/*...*/
    apply(data, new_state, changes_in_state){
        let newData = {...data, ...state};
        /* do synchrone interpolation & remaps in newData here */
        return newData;
    }
/*...*/

```

 - In an async way :
    - using this.push(newData) or by returning a referenced hashmap
    - and mostly by using wait() & release() paired functions, which will keep the store unstable and lock its update propagation until all wait() calls becomes released.


```jsx

/*...*/
    apply(data={}, new_state, changes_in_state){
        let {async1={}, async2={}, async3={}} = data;
        /* do async fill / update the data */
        this.wait('anAsyncRabbit')
        doSomeAsync(
            (err,res)=>{
                async1.datum=err||res;
                this.release('anAsyncRabbit')
            }
        )

        return {async1, async2, async3};
    }
/*...*/
    apply(data={}, new_state, changes_in_state){
        this.wait('anAsyncRabbit')
        doSomeAsync(
            (err,res)=>{
                this.push({async1:res, status:err||'ok'};
                this.release('anAsyncRabbit')
            }
        )

        return data;
    }
/*...*/

```


## Store's Workflow

- A Store have it's state updated ( action has pushed state update or a source store had its data updated )
- If this state have the required & followed value
- The apply function is called push new data in an async or sync way
- The store is stabilized and (if there is new data) propagated
- listening stores have theirs state updated and we go to step 1 until the whole context is stable

## Contexts

Rescope stores find theirs source stores in a context object, its parents and/or its mixed contexts.<br>
When a store became unstable, its context became unstable too.<br>
When a context became unstable, its parent became unstable too unless it have "rootEmitter:true" in its conf.<br>
When store or context became stable unstable they emit "stable" & "unstable" events<br>
When all child contexts of a context became stable, including itself; a context emit a "stableTree" event<br>

## Stores initial state

A store initialized with data will be stable synchronously when instantiated. <br>
If it only have a state but no data, the apply function will be called at by the constructor synchronously. (this imply that the Store object may not be fully initialized) <br>

## Actions & mutations

As the store stay independents, they deal with theirs own perimeters. <br>
The app state could be mutated using different methods depending the needs.

### Using setState

All stores inherit the setState method. <br>
Once a store state is updated, the changes are automatically propagated to the concerned stores, updating the whole app.

### Using stores functions

The stores could be enhanced with functions & setters, that will ultimately update theirs state-data pairing.

```jsx
class AppState extend Store{
        static use     = ["!AppConfig"];// require AppConfig to be applied & propagated
        static data    = {};
        switchTodoList(todoUrl){
             this.setState({todoUrl})
             // or
             this.wait();
             doSomeAsync(()=>{
                this.state.stateChange = "stand"
                this.data.
             })
        }

    }
```

### Using actions

Actions could be dispatched from contexts or directly on the stores.
* dispatching actions on contexts will trigger store's actions starting from the top parent store

```jsx
class AppState extend Store{
        static use     = ["!AppConfig"];// require AppConfig to be applied & propagated
        static actions = {
            activateSalt(arg){// binded on the store instance
                // return some state updates
                return {some:'mutations'};
                // or
                return; // to not change the state
                // wait, release, setState & push stays callable
            }
        }
    }
```

### push

Using push will update & propag the data of a store.
* This should be used with cautious as it could break the state-data coherence. (that said not all the stores needs to be predictable)

## Stores state & data serialization / restoration

Serialization & restoration is managed by the Contexts objects.<br>
Stores only have to maintain the state-data coherence, but can have initial state and data from different sources :.<br>


```jsx
class MyStore extends Store {
        static state = {};// initial state
        static data = {};// initial data (soft cloned)
};

let MyStoreInstance = new MyStore(
        BaseContext,
        {
            state : {}, // take static defined state/data precedence
            data  : {}
        }
)

```

## How to add dependencies in a store

```jsx
export default class currentUser extends Store {
        static use = ["appState", "session"];// here the source store that should be in the store context

        apply( data, { appState, session }, changes ) {
            /*...*/
            return data;
        }
};
```

## How to "remap" dependencies value & sub-values in the state/data

```jsx
export default class myInterpolatedDataStore extends Store {
        static use = {
                "someSource.someValue"          : "mySwitchValue",
                "someSource2.someStuff.value"   : "mySwitchValue2",
                "someSource3.someValue"         : "mySwitchValue3",
                "someSource4.someValue"         : "mySwitchValue4"
        };

        apply( data, { mySwitchValue, mySwitchValue2, mySwitchValue3, mySwitchValue4 }, changes ) {
            /*...*/
            return data;
        }
};
```

## How to keep a store unstable until some stores / value is initialized

```jsx
export default class myInterpolatedDataStore extends Store {

        static use = {
                "!someSource.someValue"  : "mySwitchValue",// require someSource.someValue != false
                "!someRequieredSource"   : true,
                "someSource2"            : "someSource2"
        };

        apply( data, { mySwitchValue, mySwitchValue2, mySwitchValue3, mySwitchValue4 }, changes ) {
            /*...*/
            return data;
        }
};
```

## How to only call apply & update the store if specific changes occurs in the sources store

```jsx
export default class myInterpolatedDataStore extends Store {

        static use = {
                "!someSource.someValue"  : "mySwitchValue",// require someSource.someValue != false
                "!someRequieredSource"   : true,
                "someSource2"            : "someSource2"
        };

        static follow = {// only call "apply" if one of these state keys has change
            "someSource2":(newData)=>returnTrueIfApplicable(newdata),
            "mySwitchValue":true, // just change

        }

        apply( data, { mySwitchValue, mySwitchValue2, mySwitchValue3, mySwitchValue4 }, changes ) {
            /*...*/
            return data;
        }
};
```
