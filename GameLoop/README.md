
###v0.1.4

**Contributions, Issues, Feature Requests**
[on GitHub] (https://github.com/dathinaios/gameloop##github.com/dathinaios/gameloop/issues)

# Installation

GameLoop is available as a Quark. In superCollider run `Quarks.gui` for the Quarks interface.
After installation start SuperCollider, press `Cmd-D` to get the help browser and search for GameLoop


# GameLoop Quick Start Guide

**NOTE: This guide is provided here as a quick look at the library but you should follow it from within SuperCollider where you can evaluate the examples.**

---

To run these examples you will need to:

* Install the `VectorSpace` and `MathLib` Quarks. Evaluate the following to install (note that Quarks - until the 3.7 version of SC is out at least - need SVN to be installed):

        Quarks.install("VectorSpace");
        Quarks.install("MathLib");

* Get the latest SuperCollider Plugins [here.](http://sourceforge.net/projects/sc3-plugins/files/)

These are the resources you need to run the examples. For a full list of resources and instructions see the first note at `The GameLoop Library` help file.

---


## Basics

We will start simple so be patient until we reach the interesting sounding part. We will need a decoder for the sound output.You can choose from a selection of different decoders but for now let's use the simple stereo decoder from the AmbIEM Quark:

---

NOTE: In GameLoop you can choose between the AmbIEM (3d order Ambisonic to Binaural) and Ambisonic Toolkit (First order Ambisonics - Stereo, Binaural and other configurations). Regarding Binaural the Ambisonic Toolkit produces less artifacts but AmbIEM delivers (according to my subjective comparison) a superior spatial image. The newStereo decoder used here is inferior to both but it does not require you to download the Kernels for ATK or the HRTF's for AmbIEM. See the `GameLoopDecoder` help file if you want to try other decoder configurations.

---

    GameLoopDecoder.newStereo;

Now we need to create the environment for our sounds to live in. Let's assign it to the environment variable `~gameloop`:

    (
    ~gameloop = GameLoop(40, 40, 1).play(0.05);
    ~gameloop.gui;
    )

---

NOTE: The visuals are intended as a guidance. The environment is meant to be experienced aurally.

---

Now we will add a unit to our environment. A unit is a convenience class that is comprised of an entity and one or more representations. For now we are not going to worry about the particulars. It's enough to know that the `SVUnit` stands for "Sound and Visual Unit" and that it creates for us an entity, a sound representation and a visual representation. At the most basic level an entity has a radius that describes its size and a position in space.


    (
    ~unit1 = SVUnit(
      ~gameloop,
      position: RealVector2D[18,22],
      radius: 0.4
    );
    )

Not very exciting but it is a start. Since we did not specify any input we can hear the default Impulse based sound (we are at the center of the space). Since the space is 40 x 40 the center is (20,20). The sound appears 2 metres left and 2 metres forward. Notice that we had to pass the environment(~gameloop) to the `SVUnit` so that it knows where to add itself. Also notice the number of representations on the visualisation counter. To remove the entity:

    ~unit1.remove;

## Motion and Interaction

Still working with the default sound let's make it move around.

    (
    ~unit1 = SVUnit(
      ~gameloop,
      position: RealVector2D[18, 22],
      radius: 0.4,
      maxSpeed: 10.0
    );

    ~steeringBehavior = Arrive(~unit1.entity, RealVector2D[20, 20.5]);

    ~unit1.entity.force_({ arg entity;
      ~steeringBehavior.value;
    });
    )

The `maxSpeed` argument limits the maximum speed of the sound. The motion of the `SVUnit` is defined by calling the `.force_` method on its contained entity. The method expects a function that is passed the entity. In this case that force is calculated by the `Arrive` Steering behavior. This behavior moves the entity from one place to the other, in this case from the starting position (18,22) to (20, 20.5).

We can change the target position of `Arrive` on the fly. Evaluate this a few time to move the entity to random points.

    (
    ~steeringBehavior.targetPos = RealVector2D[rrand(15.0, 24.0), rrand(15.0, 25.0)];
    )

    ~unit1.remove;

We are going to have a look at the rest of the Steering behaviors a bit later. For now let's see how the entities can interact with each other. By default entities do not interact but this can be changed by setting their `collisionType` to `\static` or `\mobile` (the default is `\free`). `\mobile` entities interact with every other entity while `\static` entities only interact with `\mobile` ones. This distinction is useful for saving processing power by avoiding unneeded collision checks.

Let's make two of our default sounding units collide.

    (
    ~unit1 = SVUnit(
      ~gameloop,
      position: RealVector2D[25, 20.5],
      radius: 0.4,
      maxSpeed: 10.0
    );

    ~unit1.entity.collisionType_(\mobile);

    ~steeringBehavior1 = Arrive(~unit1.entity, RealVector2D[18, 20.5]);

    ~unit1.entity.force_({ arg entity;
      ~steeringBehavior1.value;
    });

    ~unit2 = SVUnit(
      ~gameloop,
      position: RealVector2D[15, 20.5],
      radius: 0.4,
      maxSpeed: 10.0
    );

    ~unit2.entity.collisionType_(\mobile);

    ~steeringBehavior2 = Arrive(~unit2.entity, RealVector2D[22, 20.5]);

    ~unit2.entity.force_({ arg entity;
      ~steeringBehavior2.value;
    });

    )

Although we can see the visual feedback of the collision nothing happens in terms of sound. The reason is that if we want a result from a collision we also have to define the `collisionFunc` argument of the entities contained in our `SVUnits`. Here is a simple example.

    (
    //first let's move the units to a different position
    ~steeringBehavior1.targetPos = RealVector2D[25, 20.5];
    ~steeringBehavior2.targetPos = RealVector2D[15, 20.5];
    )

    (
    //define the collision functions for the entities
    ~unit1.entity.collisionFunc_({ arg entity, entList; entity.remove; entList.do{arg i; i.remove}});
    ~unit2.entity.collisionFunc_({ arg entity, entList; entity.remove; entList.do{arg i; i.remove}});
    );

    (
    //And let's make them collide
    ~steeringBehavior1.targetPos = RealVector2D[18, 20.5];
    ~steeringBehavior2.targetPos = RealVector2D[22, 20.5];
    )

The `collisionFunc` argument is passed a function that expects two arguments: 1) the entity itself 2) a list of the objects that our entity is colliding with. In the above example we simply removed all the units involved in the collision.

## Custom Inputs

GameLoop is using JitLib NodeProxies. The input of a `SVUnit` is defined in the input argument. The argument expects a function that is passed the current speed as an argument. This is very important as it allows us to shape the sound accordingly. As we've already seen if nothing is provided the input defaults to 'Jack the Mosquito' but let's try as an input a slightly modified example from `PlayBuf`'s help file.

    // read a sound file into memory
    b = Buffer.read(s, Platform.resourceDir +/+ "sounds/a11wlk01.wav"); // remember to free the buffer later.

    (
    ~unit1 = SVUnit(
      ~gameloop,
      position: RealVector2D[25, 20.5],
      radius: 0.4,
      maxSpeed: 5.0,
      input: {arg speed; var trig, bufnum, in;
        bufnum = b.bufnum;
        //speed is used to modulate the rate of the trigger and the startPosition
        trig = Impulse.ar(speed.linlin(0.0, 10.0, 15.0, 60.0 ));
        in = PlayBuf.ar(1, bufnum, BufRateScale.kr(bufnum), trig, speed.linlin(0, 10.0, 5000, BufFrames.kr(bufnum)), 0)*0.7;
        Decay2.ar(trig, 0.01, 0.1, in);
      }
    );

    ~steeringBehavior1 = Arrive(~unit1.entity, RealVector2D[18, 20.5]);

    ~unit1.entity.force_({ arg entity;
      ~steeringBehavior1.value;
    });

    )

    // Like before we can change the forceFunc to move it around.

    ~steeringBehavior1.targetPos = RealVector2D[rrand(15.0, 24.0), rrand(15.0, 25.0)];
    Navigating the Environment

We can move around the virtual space by adding a camera.

    (
      ~camera = CameraUnit(
        ~gameloop,
        position: RealVector2D[10, 10]
      );
    )

Focus the viusalisation window by clicking on it and use the left/right arrows to rotate and front/back to move forward and backwards. You can click the add Fence button in the GUI to get a better sense of the boundaries of the space.

Remove the camera.

    ~gameloop.removeCamera;

## More on Steering Behaviors

Let's now have a look at the rest of the available steering behaviors. I am going to provide a brief demonstration. For details about individual paramateres refer to the help files for each behavior.

A very useful steering behavior is the `PathFollowing` steering behavior. With it we can define a path for our unit to follow. You can think of it as a kind of automation but instead of defining exact movement you are guiding the sound towards a path which it follows based on its internal characteristics (maxSpeed, mass etc).

    (
    ~path = Path([RealVector2D[15, 24], RealVector2D[23, 23], RealVector2D[17, 17], RealVector2D[20, 21]]);
    ~pathFollowing= PathFollowing(~unit1.entity, ~path);

    ~unit1.entity.force_({ arg entity;
      ~pathFollowing.value;
      });
    )

Notice how the unit stops at the last location. The path takes a second argument that allows us to close the path into a loop. By default it is set to false but let's change it to see what happens.

    ~path.loop = true;

We can change the loop on the fly. Let's define a random one with the loop parameter set to true.

    (
    ~path = Path(Array.fill(rrand(2.0, 38.0), {RealVector2D[rrand(5, 35.0), rrand(5.0, 35.0)]}), true);
    ~pathFollowing= PathFollowing(~unit1.entity, ~path);
    )

The Steering behavior `Seek` works in the same way as `Arrive` but does not deccelarate when reaching its destination. This makes sense when our unit is chasing something.

    (
    ~seek = Seek(~unit1.entity, RealVector2D[18, 20.5]);
    ~unit1.entity.force_({ arg entity;
      ~seek.value;
    });
    )

Let's now change the seek value. Notice how the unit moves aroung the spot instead of stopping at it.

    ~seek.targetPos = RealVector2D[rrand(15.0, 24.0), rrand(15.0, 25.0)];

Another useful steering behavior is `Wander` that allows us to send our unit in a random but natural looking walk. Before we start though let's make our unit a bit slower to have a chance to observe it before it wanders out of the screen.

    ~unit1.entity.maxSpeed = 1.2;

And here is the example:

    (
    ~wander = Wander(~unit1.entity, 2.2, 4, 2);
    ~unit1.entity.force_({ arg entity;
      ~wander.value;
      });
    )

Bear in mind that steering behaviors can be combined. At the moment a very simple implementation has been realised with the `.sum` method of `ForceManager`. In the future I am hoping to add some more advanced methods for combining steering behaviors.

