# elm-units

> Simple, safe and convenient unit types and conversions for Elm

*Note*: This package has not yet been published!

`elm-units` is useful whenever you want to store, pass around, convert between,
compare, or do arithmetic on:

  - Durations (seconds, milliseconds, hours...)
  - Angles (degrees, radians, turns...)
  - Pixels (whole or fractional)
  - Lengths (meters, feet, inches, miles, light years...)
  - Temperatures (Celsius, Fahrenheit, kelvins)
  - Speeds (pixels per second, miles per hour...) or any other rate of change
  - Any of the other built-in quantity types: areas, accelerations, masses,
    forces, pressures, currents, voltages...
  - Or even values in your own custom units, such as 'number of tiles' in a
    tile-based game

It allows you to create data types like

```elm
type alias Camera =
    { manufacturer : String
    , fieldOfView : Angle
    , shutterSpeed : Duration
    , minimumOperatingTemperature : Temperature
    }
```

and functions like

```elm
canOperateAt : Temperature -> Camera -> Bool
canOperateAt temperature camera =
    temperature |> Quantity.greaterThan camera.minimumOperatingTemperature

{-| Compute the time necessary to cover a given distance, starting from rest,
with the given acceleration.
-}
timeToCover : Length -> Acceleration -> Duration
timeToCover distance acceleration =
    ...
```

which then let you write readable, type-safe code using whatever units you want,
with any necessary unit conversions happening automatically:

```elm
camera : Camera
camera =
    { manufacturer = "Kodak"
    , fieldOfView = Angle.degrees 60
    , shutterSpeed = Duration.milliseconds 2.5
    , minimumOperatingTemperature = Temperature.celsius -35
    }

isSafe : Bool
isSafe =
    camera |> canOperateAt (Temperature.fahrenheit -10)

quarterMileTime : Float
quarterMileTime =
    timeToCover (Length.miles 0.25) (Acceleration.metersPerSecondSquared 4.5)
        |> Duration.inSeconds
```

## Table of Contents

  - [Installation](#installation)
  - [Usage](#usage)
    - [Fundamentals](#fundamentals)
    - [Arithmetic and Comparison](#arithmetic-and-comparison)
    - [Custom Functions](#custom-functions)
    - [Custom Units](#custom-units)
    - [Understanding Quantity Types](#understanding-quantity-types)
  - [Getting Help](#getting-help)
  - [API](#api)
  - [Contributing](#contributing)
  - [License](#license)

## Installation

Once this package is published, you will be able to install it with

```
elm install ianmackenzie/elm-units
```

## Usage

### Fundamentals

To take code that currently uses raw `Float` values and convert it to using
`elm-units` types, there are three basic steps:

  - Wherever you store a `Float`, such as in your model or in a message, switch
    to storing a `Duration` or `Angle` or `Temperature` etc. value instead.
  - Whenever you *have* a `Float` (from an external package, JSON decoder etc.),
    use a function such as `Duration.seconds`, `Angle.degrees` or
    `Temperature.fahrenheit` to turn it into a type-safe value.
  - Whenever you *need* a `Float` (to pass to an external package, encode as
    JSON etc.), use a function such as `Duration.inMillliseconds`,
    `Angle.inRadians` or `Temperature.inCelsius` to extract the value in
    whatever units you want.

Let's see what that looks like in practice! A pretty common pattern in Elm apps
is to have a `Tick Float` message that contains a number of milliseconds since
the last animation frame, but it's very easy to accidentally treat that `Float`
as a number of seconds, leading to things like way-too-fast animations. Let's
modify the code to use a `Duration` value instead.

First, define the `Tick` message to store a `Duration` value instead of a
`Float`, then take the value returned by the `onAnimationFrameDelta`
subscription and convert it to a `Duration` using the `milliseconds` function:

```elm
import Browser.Events
import Duration exposing (Duration)

type Msg
    = Tick Duration

subscriptions model =
    -- Convert the Float value to a Duration using the
    -- 'milliseconds' function, then store in a Tick message
    Browser.Events.onAnimationFrameDelta (Duration.milliseconds >> Tick)
```

Later on, when you handle the `Tick` message, you can extract the duration value
in whatever units you want. Perhaps you want to keep track of how many frames
per second your application is running at:

```elm
update message model =
    case message of
        Tick duration ->
            -- Extract the duration as a number of seconds
            ( { model | fps = 1 / Duration.inSeconds duration }
            , Cmd.none
            )
```

Note that any necessary conversion is done automatically - you create a
`Duration` from a `Float` by specifying what kind of units you *have*, and then
later extract a `Float` value by specifying what kind of units you *want*. This,
incidentally, means you can skip the 'store the value in a message' step and
just use the provided functions to do unit conversions:

```elm
Duration.hours 3 |> Duration.inSeconds
--> 10800

Length.feet 10 |> Length.inMeters
--> 3.048

Speed.milesPerHour 60 |> Speed.inMetersPerSecond
--> 26.8224

Temperature.celsius 30 |> Temperature.inFahrenheit
--> 86
```

If you try to do a conversion that doesn't make sense, you'll get a compile
error:

```elm
Duration.seconds 30 |> Length.inMeters
-- TYPE MISMATCH ----------------------------------------------------------- elm

This function cannot handle the argument sent through the (|>) pipe:

1|   Duration.seconds 30 |> Length.inMeters
                            ^^^^^^^^^^^^^^^
The argument is:

    Duration

But (|>) is piping it a function that expects:

    Length
```

### Arithmetic and Comparison

You can do basic math with units:

```elm
-- 6 feet 3 inches, converted to meters
Length.feet 6 |> Quantity.add (Length.inches 3) |> Length.inMeters
--> 1.9050000000000002

-- pi radians plus 45 degrees is 5/8 of a full turn
Quantity.sum [ Angle.radians pi, Angle.degrees 45 ] |> Angle.inTurns
--> 0.625

-- Area of a triangle with base of 2 feet and height of 8 inches
Quantity.product (Length.feet 2) (Length.inches 8)
    |> Quantity.scaleBy 0.5
    |> Area.inSquareInches
--> 96
```

Special support is provided for calculations involving rates of change:

```elm
-- How long do we travel in 10 seconds at 100 km/h?
Duration.seconds 10
    |> Quantity.at (Speed.kilometersPerHour 100)
    |> Length.inMeters
--> 277.77777777777777

-- How long will it take to travel 20 km if we're driving at 60 mph?
Length.kilometers 20
    |> Quantity.at_ (Speed.milesPerHour 60)
    |> Duration.inMinutes
--> 12.427423844746679

-- How fast is "a mile a minute", in kilometers per hour?
Length.miles 1 |> Quantity.per (Duration.minutes 1) |> Speed.inKilometersPerHour
--> 96.56064

-- Reverse engineer the speed of light from defined lengths/durations
speedOfLight =
    Length.lightYears 1 |> Quantity.per (Duration.years 1)

speedOfLight |> Speed.inMetersPerSecond
--> 299792458

-- One astronomical unit is the (average) distance from the Sun to the Earth
-- Roughly how long does it take light to reach the Earth from the Sun?
Length.astronomicalUnits 1 |> Quantity.at_ speedOfLight |> Duration.inMinutes
--> 8.316746397269274
```

Note that the various functions above are not restricted to speed (length per
unit time) - any units work:

```elm
pixelsPerInch =
    Pixels.pixels 96 |> Quantity.per (Length.inches 1)

Length.centimeters 3 |> Quantity.at pixelsPerInch |> Pixels.inPixels
--> 113.38582677165354
```

Finally, `Quantity` values can be compared/sorted:

```elm
meters 1 |> Quantity.greaterThan (feet 3)
--> True

Quantity.compare (meters 1) (feet 3)
--> GT

Quantity.max (meters 1) (feet 3)
--> meters 1

Quantity.maximum [ meters 1, feet 3 ]
--> Just (meters 1)

Quantity.sort [ meters 1, feet 3 ]
--> [ feet 3, meters 1 ]
```

### Custom Functions

Some calculations cannot be expressed using the built-in `Quantity` functions.
Take kinetic energy `E_k = 1/2 * m * v^2`, for example - the `elm-units` type
system is not sophisticated enough to work out the units properly. Instead,
you'd need to create a custom function like

```elm
kineticEnergy : Mass -> Speed -> Energy
kineticEnergy (Quantity m) (Quantity v) =
    Quantity (0.5 * m * v^2)
```

In the _implementation_ of `kineticEnergy`, you're working with raw `Float`
values so you need to be careful to make sure the units actually do work out.
(The values will be in [SI](https://en.wikipedia.org/wiki/International_System_of_Units)
units - meters, seconds etc.) Once the function has been implemented, though, it
can be used un a completely type-safe way - callers can supply arguments using
whatever units they like, and extract results in whatever units they want:

```elm
kineticEnergy (Mass.tonnes 1.5) (milesPerHour 60) |> inKilowattHours
--> 0.14988357119999998
```

### Custom Units

`elm-units` defines many standard unit types, but you can easily define your own! See [CustomUnits](doc/CustomUnits.md) for an example.

### Understanding Quantity Types

The same quantity type can often be expressed in multiple different ways, which is important to understand especially when trying to interpret error messages! Take the `Speed` type alias as an example. It is defined as

```elm
type alias Speed =
    Fractional MetersPerSecond
```

Expanding the `MetersPerSecond` type alias, this is

```elm
Fractional (Quotient Meters Seconds)
```

but there also exists a generic `Rate` type alias

```elm
type alias Rate dependentUnits independentUnits =
    Fractional (Quotient dependentUnits independentUnits)
```

which means `Speed` can be expressed as

```elm
Rate Meters Seconds
```

Alternately, expanding the `Fractional` type alias gives

```elm
Quantity Float (Quotient Meters Seconds)
```

which is the "true" type with no type aliases left to expand.

## Getting Help

For general questions about using `elm-units`, try asking in the [Elm Slack](http://elmlang.herokuapp.com/)
or posting on the [Elm Discourse forums](https://discourse.elm-lang.org/) or the
[Elm subreddit](https://www.reddit.com/r/elm/). I'm **@ianmackenzie** on all
three platforms =)

## API

Full API documentation will be available on the Elm package web site once this
package is published.

## Contributing

TODO

## License

[BSD-3-Clause © Ian Mackenzie](LICENSE)
