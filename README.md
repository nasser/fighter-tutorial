# Arcadia for Clojure Programmers

This tutorial assumes familiarity with Clojure, and a basic understanding of Unity's scene graph and messaging system.

This is bare-bones, out-of-the-box Arcadia. Many libraries exist that extend its functionality further.

## Overview of the Arcadia Role System

Arcadia provides an opt-in bridge to the scene graph, representing bundles of state and behavior as persistent maps called 'roles'. The `:state` entry holds data, and the other entries associate Unity [messages](https://docs.unity3d.com/Manual/EventFunctions.html) (also known as "event functions") with `IFn` instances, encoded as `:update`, `:fixed-update`, `:on-collision-enter`, etc (the complete list can be found in the `arcadia.core/hook-types` map). When a message is dispatched to the GameObject, any `IFn`s associated with that message via an attached role will be called.

Here's an example of a role:

```clojure
(def example-role
  {:state {:health 10} ; data to attach to the object
   :update #'some-update-function ; var, the function value of which will run during the Update message
   :on-collision-enter #'some-collision-function} ; var, the function value of which will run during a collision event
   )
```

This role would be attached to a GameObject `obj` like this:

```clojure
(role+ obj ::example-role example-role)
```

Here the keys `:update` and `:on-collision-enter` correspond to the `Update` and `OnCollisionEnter` Unity messages. Their values are Clojure vars that will be invoked in response to those Unity messages. The value associated with `:update`, in this case the var `#'some-update-function`, will run every frame, because Unity triggers the `Update` message every frame. Similarly, the the value associated with `:on-collision-enter`, here the var `#'some-collision-function`, will run when any GameObject this role is attached to collides with something. The keyword `::example-role` in `role+` is called the _role key_, and is used to look up the `:state` associated with this particular role.

Anything implementing the `IFn` interface for the correct arity is supported as a value for the message entries; that is,

```clojure
{:update (fn [obj k] (UnityEngine.Debug/Log "running update"))}
```

would also work. Clojure vars, which also implement `IFn`, are greatly preferred, however, because they can be dynamically redefined from the REPL, and can [_serialize_](https://docs.unity3d.com/560/Documentation/Manual/script-Serialization.html).

The parameters expected of the `IFn` associated with a key are determined by the parameters expected of the corresponding Unity event function. A callback should have the same parameters as the corresponding Unity event function, plus an additional first parameter for the GameObject itself, and an additional final parameter for the key.

For example, the signature of the [OnCollisionEnter](https://docs.unity3d.com/ScriptReference/MonoBehaviour.OnCollisionEnter.html) message is
```
GameObject.OnCollisionEnter(Collision)
```
The signature of a function associated with the OnCollisionEnter message via `:on-collision-enter` is therefore
```
(fn [^GameObject obj, ^UnityEngine.Collision collision, role-key] ...)
```

To take another example, the signature of the [Update](https://docs.unity3d.com/ScriptReference/MonoBehaviour.Update.html) message is

```
GameObject.Update()
```

That is, there are no parameters. The expected signature of a function associated with the Update message via `:update` is therefore

```
(fn [^GameObject obj, role-key] ...)
```

## Building the player's avatar

First, we define the `ns` form to set up the namespace.

```clojure
(ns fighter-tutorial.core
  (:use arcadia.core arcadia.linear)
  (:import [UnityEngine Collider2D Physics
            GameObject Input Rigidbody2D
            Vector2 Mathf Resources Transform
            Collision2D Physics2D]
           ArcadiaState))
```

Let's start by making an inert GameObject representing the player. We'll do this by [instantiating](https://docs.unity3d.com/Manual/InstantiatingPrefabs.html) the `"fighter"` [prefab](https://docs.unity3d.com/Manual/Prefabs.html).

```clojure
(defonce player-atom
  (atom nil))

(defn setup []
  (when @fighter-atom (retire @player-atom)) ; `retire` removes a GameObject from the scene
  (let [player (GameObject/Instantiate (Resources/Load "fighter"))]
    (reset! player-atom player)))
```

After evaluating this code, run `(setup)` in the REPL. A new GameObject should appear, looking like this:

If we call `(setup)` multiple times at this point, the scene will seem to remain the same, but really we're destroying and recreating the player every time.

Now let's define some helper functions for input and math.

```clojure
(defn bearing-vector [angle]
  (let [angle (* Mathf/Deg2Rad angle)]
    (v2 (Mathf/Cos angle) (Mathf/Sin angle))))

(defn abs-angle [v]
  (* Mathf/Rad2Deg
     (Mathf/Atan2 (.y v) (.x v))))

(defn controller-vector []
 (v2 (Input/GetAxis "Horizontal")
     (Input/GetAxis "Vertical")))

(defn wasd-key []
  (or (Input/GetKey "w")
      (Input/GetKey "a")
      (Input/GetKey "s")
      (Input/GetKey "d")))
```

### Player Movement

Now we can write the interactive movement logic.

To review, Arcadia associates state and behavior with GameObjects using maps called _roles_. Roles are attached to GameObjects on a key called the _role key_.

Roles specify callbacks that run in response to Unity messages, as well as an optional `:state` entry that holds data.

```clojure
(defn player-movement-fixed-update [obj k] ; We'll only use the `obj` parameter
  (with-cmpt obj [rb Rigidbody2D]          ; Gets the Rigidbody2D component
    (when (wasd-key)                       ; Checks for WASD key
      (.MoveRotation rb (abs-angle (controller-vector))) ; Rotates towards key
      (.AddForce rb                                      ; Moves forwards
        (v2* (bearing-vector (.rotation rb))
             3)))))

;; Associates the FixedUpdate Unity message with a var in a "role" map
(def player-movement-role
  {:fixed-update #'player-movement-fixed-update})
```

Roles, in turn, can be gathered together into maps, and the roles in these maps attached to GameObjects with `roles+`.

```clojure
;; Packages the role up in a map with a descriptive key
(def player-roles
  {::movement player-movement-role})
```

Note that there is no deep need to have a separate `player-movement-role` var, for our purposes here the following would work just as well:

```clojure
(def player-roles
  {::movement {:fixed-update #'player-movement-fixed-update}})
```

Finally, we modify the `setup` function to attach the state and behavior specified in `player-roles`. We'll continue to modify this function as we add features to the game.

```clojure
(defn setup []
  (when @fighter-atom (retire @fighter-atom))
  (let [fighter (GameObject/Instantiate (Resources/Load "fighter"))]
    (reset! fighter-atom fighter)
    (roles+ fighter player-roles))) ; NEW
```

From the REPL, call `(setup)` again. Back in the Unity Game view, the player should now be controllable using the `w` `a` `s` `d` keys.

## Bullets

Now let's shoot some bullets. We want the fighter to launch a bullet every time the player hits the space key. We also want to clean up bullets after a certain amount of time.

We'll need:

- A function that "shoots" a bullet, placing it in the scene graph
- A role that moves the bullet forward every physics frame
- A role that calls the shooting function when the player hits space
- A role responsible for removing bullets after a certain period of time.

Let's start with the time restriction, so bullets don't pile up.

We could define a role for this the way we've been doing, like so:
```clojure
(defn lifespan-update [obj k]
  (let [{:keys [start lifespan]} (state obj k)]
    (when (< lifespan (.TotalMilliseconds (.Subtract System.DateTime/Now start)))
      (retire obj))))

(def lifespan-role
  {:state {:start System.DateTime/Now
           :lifespan 0}
   :update #'lifespan-update})
```

This can get a little tedious, however. Arcadia provides a `defrole` macro to speed the process of defining roles.

```clojure
(defrole lifespan-role
  :state {:start System.DateTime/Now
          :lifespan 0}
  (update [obj k]
    (let [{:keys [start lifespan]} (state obj k)]
      (when (< lifespan (.TotalMilliseconds (.Subtract System.DateTime/Now start)))
        (retire obj)))))
```

This is exactly equivalent to:

```clojure
(defn lifespan-role-update [obj k]
  (let [{:keys [start lifespan]} (state obj k)]
    (when (< lifespan (.TotalMilliseconds (.Subtract System.DateTime/Now start)))
      (retire obj))))

(def lifespan-role
  {:state {:start System.DateTime/Now
           :lifespan 0}
   :update #'lifespan-update})
```

We'll use `defrole` from now on.

We want to avoid bullet self-collision. In Unity we can do this by setting the `"bullets"` layer to avoid collisions with itself. Modify `setup` to do so:

```clojure
(defn setup []
  (let [bullets (UnityEngine.LayerMask/NameToLayer "bullets")]         ; NEW
    (Physics2D/IgnoreLayerCollision (int bullets) (int bullets) true)) ; NEW
  (when @player-atom (retire @player-atom))
  (let [player (GameObject/Instantiate (Resources/Load "fighter"))]
    (set! (.name player) "player")
    (roles+ player player-roles)
    (reset! player-atom player)))
```

Bullet movement is just:

```clojure
(defrole bullet-movement-role
  (fixed-update [bullet k]
    (with-cmpt bullet [rb Rigidbody2D]
      (move-forward rb 0.2))))
```

Finally, the roles map for bullets:

```clojure
(def bullet-roles
  {::movement bullet-movement-role
   ::restrict-distance restrict-distance-role})
```

We would like to share the shooting logic with both the player and non-player entities. We'll use two functions, `shoot` and `shooter-shoot`. `shoot` takes a `UnityEngine.Vector2` starting position `start` and an angle `bearing`, and creates a new bullet at that position and angle, returning the bullet.

`shooter-shoot` takes a GameObject and `shoot`s a bullet forward from it, set to ignore collisions with the GameObject itself. [RAMSEY: is this necessary for triggers?]

```clojure
(defn shoot [start bearing]
  (let [bullet (GameObject/Instantiate (Resources/Load "missile" GameObject))]
    (with-cmpt bullet [rb Rigidbody2D
                       tr Transform]
      (set! (.position tr) (v3 (.x start) (.y start) 1))
      (.MoveRotation rb bearing))
    (roles+ bullet
      (-> bullet-roles
          (assoc-in [::restrict-distance :state :start] start)))
    bullet))

(defn shooter-shoot [obj]
  (with-cmpt obj [rb Rigidbody2D]
    (let [bullet (shoot (.position rb) (.rotation rb))]
      (do-ignore-collisions (cmpt obj Collider2D) (cmpt bullet Collider2D))
      bullet)))
```

Now we give the player the ability to shoot bullets by hitting space:

```clojure
(defrole player-shooting-role
  (fixed-update [obj k]
    (with-cmpt obj [rb Rigidbody2D]
      (when (Input/GetKeyDown "space")
        (shooter-shoot obj)))))
```

We add this functionality to the player by going back and editing `player-roles`:

```clojure
(def player-roles
  {::movement player-movement-role
   ::shooting player-shooting-role}) ; NEW
```

## The enemy

We can create the enemy using the same technique: define roles, attach them to a GameObject. Note the reuse of `shooter-shoot` and `health-role`.

```clojure
;; villain shooting
(defrole villain-shooting-role
  :state {:last-shot System.DateTime/Now
          :target nil}
  (update [obj k]
    (let [{:keys [target last-shot]} (state obj k)
          now System.DateTime/Now]
      (when (and target (< 1000 (.TotalMilliseconds (.Subtract now last-shot))))
        (update-state obj k assoc :last-shot now)
        (shooter-shoot obj)))))

;; villain movement
(defrole villain-movement-role
  :state {:target nil}
  (fixed-update [obj k]
    (let [{:keys [target]} (state obj k)]
      (when (not (null-obj? target))
        (with-cmpt obj [rb1 Rigidbody2D]
          (with-cmpt target [rb2 Rigidbody2D]
            (let [pos-diff (v2- (.position rb2) (.position rb1))
                  rot-diff (Vector2/SignedAngle
                             (bearing-vector (.rotation rb1))
                             pos-diff)]
              (.MoveRotation rb1
                (+ (.rotation rb1)
                   (Mathf/Clamp -1 rot-diff 1))))))))))

(def villain-roles
 {::shooting villain-shooting-role
  ::movement villain-movement-role})

;; function to construct the villain
(defn make-villain [protagonist]
 (let [villain (GameObject/Instantiate (Resources/Load "villain" GameObject))]
   (roles+ villain
     (-> villain-roles
         (assoc-in [::shooting :state :target] protagonist)
         (assoc-in [::movement :state :target] protagonist)))))
```

Now we can add the enemy to the `setup` function:
```clojure
(defn setup []
  (let [bullets (UnityEngine.LayerMask/NameToLayer "bullets")]
    (Physics2D/IgnoreLayerCollision (int bullets) (int bullets) true))
  (when @player-atom (retire @player-atom))
  (let [player (GameObject/Instantiate (Resources/Load "fighter"))]
    (set! (.name player) "player")
    (roles+ player player-roles)
    (make-villain player) ; NEW
    (reset! player-atom player)))
```

To make the player susceptible to the enemy's bullets, we need only give it the `health-role`:

```clojure
(def player-roles
  {::movement player-movement-role
   ::shooting player-shooting-role
   ::health (update health-role :state assoc :health 10)}) ; NEW
```

To make the enemy susceptible to the player's bullets, we do the same:

```clojure
(def villain-roles
  {::shooting villain-shooting-role
   ::movement villain-movement-role
   ::health (update health-role :state assoc :health 10)}) ; NEW
```