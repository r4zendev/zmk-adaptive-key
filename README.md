# ZMK-ADAPTIVE-KEY

This module adds a `adaptive-key` behavior to ZMK. Some highlights compared to
existing alternatives:

- Works as a module without the need to patch ZMK.
- Configurable `dead-keys` property to turn any keycode into a dead key.
- Simple "inline" macro specification to bind behavior sequences.
- `min-prior-idle-ms` and `max-prior-idle` timeout properties that can vary by
  trigger.
- Correct handling of explicit modifiers.

## Usage

To load the module, add the following entries to `remotes` and `projects` in
`config/west.yml`.

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: urob
      url-base: https://github.com/urob
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3 # Set to desired ZMK release.
      import: app/west.yml
    - name: zmk-adaptive-key
      remote: urob
      revision: v0.3 # Should match ZMK release.
  self:
    path: config
```

## Configuration

An `adaptive-key` defines "trigger" conditions on the _last_ keycode pressed
prior to pressing the behavior. If any trigger condition matches, a behavior
bound to that trigger is invoked. If no trigger condition matches, a default
behavior is invoked.

### `trigger` properties

Triggers are defined as child nodes of an adapative-key instance and are checked
in order of their definition. Triggers have two _required properties_:

- **`trigger-keys`**: A list of keycodes that trigger the bindings.
- **`bindings`**: Behaviors bound to the trigger. If set to multiple behaviors
  they are invoked in sequence.

Additional conditions can be configured via _optional properties_:

- **`min-prior-idle-ms`**: Minimum time that must be elapsed since the last key
  press. Defaults to none.
- **`max-prior-idle-ms`**: Maximum time that must be elapsed since the last key
  press. Defaults to none.
- **`strict-modifiers`**: If true, modifiers must _exactly_ match the
  `trigger-keys`. Otherwise it suffices to _contain_ the `trigger-keys` (useful
  for case-sensitive bindings). Defaults to false.

### `adaptive-key` properties

Besides `trigger` child nodes, `adaptive-key` instances have the following
properties:

- **`bindings`** (required): The default behavior to invoke if no trigger
  condition is met. Can be `&none` to do nothing.
- **`dead-keys`**: A list of key codes that are converted to dead keys. Dead
  keys don't send a keycode when pressed the first time but are still considered
  as trigger condition. If pressed again, dead keys send their normal keycode.

## Examples

### Hands-down adaptive keys

```c
/ {
    behaviors {
        ak_h: ak_h {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&kp H>;

            akt_ah { trigger-keys = <A>; max-prior-idle-ms = <300>; bindings = <&kp U>; };
            akt_uh { trigger-keys = <U>; max-prior-idle-ms = <300>; bindings = <&kp A>; };
            akt_eh { trigger-keys = <E>; max-prior-idle-ms = <300>; bindings = <&kp O>; };
        };

        ak_m: ak_m {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&kp M>;

            akt_gm { trigger-keys = <G>; max-prior-idle-ms = <300>; bindings = <&kp L>; };
            akt_pm { trigger-keys = <P>; max-prior-idle-ms = <300>; bindings = <&kp L>; };
        };

        // And similarly for VP->VL, PV->LV, BT->BL, TB->LB

        ak_g: ak_g {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&kp G>;

            // Binding two behaviors: JG->JPG
            akt_jg { trigger-keys = <J>; max-prior-idle-ms = <300>; bindings = <&kp P &kp G>; };
        };

    };
};
```

### Dead keys

```c
/ {
    behaviors {
        ak_e: ak_e {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&kp E>;
            dead-keys = <GRAVE CARET APOS QUOTE>;

            grave { trigger-keys = <GRAVE>; bindings = <&fr_e_grave>; };
            acute { trigger-keys = <APOS>; bindings = <&fr_e_acute>; };
            circumflex { trigger-keys = <CARET>; bindings = <&fr_e_circumflex>; };
            diaeresis { trigger-keys = <QUOTE>; bindings = <&fr_e_diaeresis>; };
        };
    };
};
```

Note: the behavior bindings `&fr_e_grave` etc must be defined elsewhere (e.g.,
using the French
[language header](https://github.com/urob/zmk-helpers/tree/main#unicode-characters-and-language-collection)
from the `zmk-helpers` module).

Alternatively, "new" dead keycodes can be "created" by cannibalizing unused
keycode. For instance:

```c
#define DEAD1 F21
#define DEAD2 F22
#define DEAD3 F23
#define DEAD4 F24

/ {
    behaviors {
        ak_e: ak_e {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&kp E>;
            dead-keys = <DEAD1 DEAD2 DEAD3 DEAD4>;

            grave { trigger-keys = <DEAD1>; bindings = <&fr_e_grave>; };
            acute { trigger-keys = <DEAD2>; bindings = <&fr_e_acute>; };
            circumflex { trigger-keys = <DEAD3>; bindings = <&fr_e_circumflex>; };
            diaeresis { trigger-keys = <DEAD4>; bindings = <&fr_e_diaeresis>; };
        };
    };
};
```

Note: While the keycodes used in this example are typically unused, they are
still [defined](https://zmk.dev/docs/keymaps/list-of-keycodes#f-keys). Making up
new _undefined_ keycodes is unsupported as their working hinges on the execution
order of this module, which cannot be configured by any supported means.

### Shift-repeat

```c
/ {
    behaviors {
        shift-repeat: shift-repeat {
            compatible = "zmk,behavior-adaptive-key";
            #binding-cells = <0>;
            bindings = <&sk LSHFT>;

            repeat {
                trigger-keys = <A B C D E F G H I J K L M N O P Q R S T U V W X Y Z>;
                bindings = <&key_repeat>;
                max-prior-idle-ms = <350>;
                strict-modifiers;
            };
        };
    };
};
```

This sets up a `shift-repeat` behavior that sends `&sk LSHFT` unless when
pressed within 0.35 seconds of any alpha key, in which case it sends
`&key_repeat`. Great for your homing thumb key!

### Multi-key history

By default a trigger only looks at the _last_ keycode. The following optional
properties let a trigger also condition on keys typed before it.

- **`prior-trigger-keys`** (trigger property): when set, the trigger fires only
  if the _second-to-last_ keycode also matches one of these, on top of
  `trigger-keys` matching the last keycode. This AND-gates a trigger on the last
  two keys, e.g. distinguishing `ntc` from `atc`:

  ```c
  / {
      behaviors {
          ak: ak {
              compatible = "zmk,behavior-adaptive-key";
              #binding-cells = <0>;
              bindings = <&kp C>;

              // Only fire after "nt", not after "at".
              ntc { trigger-keys = <T>; prior-trigger-keys = <N>; bindings = <&kp X>; };
          };
      };
  };
  ```

- **`prior-keys`** (trigger property): an ordered sequence of keycodes that must
  precede the trigger, oldest first. The last element is the key just before the
  trigger match, the one before it the key before that, and so on. Combined with
  `trigger-keys` this matches N preceding keys, e.g. `<Y O>` + `<U>` matches the
  word `you`:

  ```c
  / {
      behaviors {
          ak: ak {
              compatible = "zmk,behavior-adaptive-key";
              #binding-cells = <0>;
              bindings = <&kp U>;

              // Matches the sequence "y", "o", "u".
              you { trigger-keys = <U>; prior-keys = <Y O>; bindings = <&macro_you>; };
          };
      };
  };
  ```

  The depth of available history is set by
  `CONFIG_ZMK_ADAPTIVE_KEY_HISTORY_DEPTH` (defaults to 6).

- **`skip-magic`** (`adaptive-key` property): when set, every trigger of the
  instance matches against the _second-to-last_ keycode instead of the last one.
  If no trigger matches, the second-to-last keycode is re-sent (skip-repeat
  fallback). Useful to act on the key before the most recent one:

  ```c
  / {
      behaviors {
          ak: ak {
              compatible = "zmk,behavior-adaptive-key";
              #binding-cells = <0>;
              bindings = <&kp SPACE>;
              skip-magic;

              trig { trigger-keys = <A>; bindings = <&kp B>; };
          };
      };
  };
  ```

## `Kconfig` settings

- `CONFIG_ZMK_ADAPTIVE_KEY_MAX_TRIGGER_CONDITIONS`: Maximum number of trigger
  conditions per `adaptive-key` behavior. Defaults to 32.
- `CONFIG_ZMK_ADAPTIVE_KEY_MAX_BINDINGS`: Maximum number of behaviors bound to a
  trigger (i.e., length of macro sequence). Defaults to 4.
- `CONFIG_ZMK_ADAPTIVE_KEY_WAIT_MS`: Wait time in milliseconds between key
  presses when binding a macro sequence. Defaults to 5ms.
- `CONFIG_ZMK_ADAPTIVE_KEY_TAP_MS`: Hold time per key tap when binding a macro
  sequence. Defaults to 5ms.
- `CONFIG_ZMK_ADAPTIVE_KEY_HISTORY_DEPTH`: Number of recent keycodes kept for
  `prior-keys` matching. Defaults to 6.

## References

- The behavior idea is inspired by the
  [Hands Down](https://sites.google.com/alanreiser.com/handsdown/home#h.3fq4ywspvw1g)
  keyboard layout. The original ZMK
  [feature request](https://github.com/zmkfirmware/zmk/issues/1624) provides
  some further discussion.
- PR [#2042](https://github.com/zmkfirmware/zmk/pull/2042) provides an
  alternative implementation.
- My personal [zmk-config](https://github.com/urob/zmk-config) contains advanced
  usage examples.
