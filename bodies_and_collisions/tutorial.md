# DART Tutorial (bodies_and_collisions)

This tutorial will show you how to programmatically create different kinds of bodies and set initial conditions for Skeletons. It will also demonstrate some use of DART’s Frame Semantics.
The tutorial consists of five Lessons covering the following topics:

* Creating a rigid body
* Setting initial conditions and taking advantage of Frames
* Setting joint spring and damping properties

## Lesson 1 Creating a rigid body
Start by going opening the code. Find the function named `addRigidBody`. 
Different Joint types in DART are managed by a bunch of different classes, so we need to use templates if we want the same function to work with a variety of Joint types.

### Lesson 1a Setting joint properties

The first thing we’ll want to do is set the Joint properties for our new body. Whenever we create a BodyNode, we must also create a parent Joint for it. A BodyNode needs a parent Joint, even if that BodyNode is the root of the Skeleton, because we need its parent Joint to describe how it’s attached to the world. A root BodyNode could be attached to the world by any kind of Joint. Most often, it will be attached by either a FreeJoint (if the body should be completely free to move with respect to the world) or a WeldJoint (if the body should be rigidly attached to the world, unable to move at all), but any Joint type is permissible.

Joint properties are managed in a nested class, which means it’s a class which is defined inside of another class. For example, `RevoluteJoint` properties are managed in a class called `RevoluteJoint::Properties` while `PrismaticJoint` properties are managed in a class called `PrismaticJoint::Properties`. However, both `RevoluteJoint` and `PrismaticJoint` inherit the `SingleDofJoint` class so the `RevoluteJoint::Properties` and `PrismaticJoint::Properties` classes both inherit the `SingleDofJoint::Properties` class. The difference is that `RevoluteJoint::Properties` also inherits `RevoluteJoint::UniqueProperties` whereas `PrismaticJoint::Properties` inherits `PrismaticJoint::UniqueProperties` instead. Many DART classes contain nested Properties classes like this which are compositions of their base class’s nested `Properties` class and their own `UniqueProperties` class. As you’ll see later, this is useful for providing a consistent API that works cleanly for fundamentally different types of classes.

To create a `Properties` class for our Joint type, we’ll want to say

```cpp
    typename JointType::Properties properties;
```
We need to include the `typename` keywords because of how the syntax works for templated functions. Leaving it out should make your compiler complain.
From here, we can set the Joint properties in any way we’d like. There are only a few things we care about right now: First, the Joint’s name. Every Joint in a Skeleton needs to have a non-empty unique name. Those are the only restrictions that are placed on Joint names. If you try to make a Joint’s name empty, it will be given a default name. If you try to make a Joint’s name non-unique, DART will append a number tag to the end of the name in order to make it unique. It will also print out a warning during run time, which can be an eyesore (because it wants you to be aware when you are being negligent about naming things). For the sake of simplicity, let’s just give it a name based off its child BodyNode:
```cpp
    properties.mName = name+"_joint";
```
Don’t forget to uncomment the function arguments.

Next we’ll want to deal with offsetting the new BodyNode from its parent BodyNode. We can use the following to check if there is a parent BodyNode:
```cpp
    Eigen::Isometry3d tf(Eigen::Isometry3d::Identity());
```
An `Eigen::Isometry3d` is the Eigen library’s version of a homogeneous transformation matrix. Here we are initializing it to an Identity matrix to start out. This is almost always something you should do when creating an Eigen::Isometry3d, because otherwise its contents will be completely arbitrary trash.

We can easily compute the center point between the origins of the two bodies using our default height value:
```cpp
    tf.translation() = Eigen::Vector3d(0, 0, default_shape_height / 2.0);
```
We can then offset the parent and child BodyNodes of this Joint using this transform:
```cpp
    properties.mT_ParentBodyToJoint = tf;
    properties.mT_ChildBodyToJoint = tf.inverse();
```
Remember that all of that code should go inside the `if(parent)` condition. We do not want to create this offset for root BodyNodes, because later on we will rely on the assumption that the root Joint origin is lined up with the root BodyNode origin.

### Lesson 1b Create a Joint and BodyNode pair
A single function is used to simultaneously create a new Joint and its child BodyNode. It’s important to note that a Joint cannot be created without a child BodyNode to accompany it, and a BodyNode cannot be created with parent Joint to attach it to something. A parent Joint without a child BodyNode or vice-versa would be non-physical and nonsensical, so we don’t allow it.
Use the following to create a new Joint & BodyNode, and obtain a pointer to that new BodyNode:
```cpp
    BodyNode* bn = chain->createJointAndBodyNodePair<JointType>(
      parent, properties, BodyNode::AspectProperties(name)).second;
```
There’s a lot going on in this function, so let’s break it down for a moment:
```cpp
  chain->createJointAndBodyNodePair<JointType>
```
This is a Skeleton member function that takes template arguments. The first template argument specifies the type of Joint that you want to create. In our case, the type of Joint we want to create is actually a template argument of our current function, so we just pass that argument along. The second template argument of `createJointAndBodyNodePair` allows us to specify the BodyNode type that we want to create, but the default argument is a standard rigid BodyNode, so we can leave the second argument blank.
```cpp
  (parent, properties, BodyNode::AspectProperties(name))
```
Now for the function arguments: The first specifies the parent BodyNode. In the event that you want to create a root BodyNode, you can simply pass in a nullptr as the parent. The second argument is a `JointType::Properties` struct, so we pass in the properties object that we created earlier. The third argument is a `BodyNode::Properties` struct, but we’re going to set the BodyNode properties later, so we’ll just toss the name in by wrapping it up in a `BodyNode::AspectProperties` object and leave the rest as default values.

The function actually returns a std::pair of pointers to the new Joint and new BodyNode that were just created, but we only care about grabbing the BodyNode once the function is finished, so we can append `.second` to the end of the line so that we just grab the BodyNode pointer and ignore the Joint pointer. The joint will of course still be created; we just have no need to access it at this point.

### Lesson 1c Make a shape for the body
We’ll take advantage of the `Shape::ShapeType` enumeration to specify what kind of Shape we want to produce for the body. In particular, we’ll allow the user to specify three types of Shapes: `Shape::BOX`, `Shape::CYLINDER`, and `Shape::ELLIPSOID`.
```cpp
  ShapePtr shape;
  if(Shape::BOX == type)
  {
    // TODO: Make a box
  }
  else if(Shape::CYLINDER == type)
  {
    // TODO: Make a cylinder
  }
  else if(SHAPE::ELLIPSOID == type)
  {
    // TODO: Make an ellipsoid
  }
```
`ShapePtr` is simply a typedef for `std::shared_ptr<Shape>`. DART has this typedef in order to improve space usage and readability, because this type gets used very often.

Now we want to construct each of the Shape types within their conditional statements. Each constructor is a bit different.

For box we pass in an Eigen::Vector3d that contains the three dimensions of the box:
```cpp
  shape = std::make_shared<BoxShape>(Eigen::Vector3d(
                                      default_shape_width,
                                      default_shape_width,
                                      default_shape_height));
```
For cylinder we pass in a radius and a height:
```cpp
shape = std::make_shared<CylinderShape>(default_shape_width/2.0,
                                        default_shape_height);
```
For ellipsoid we pass in an Eigen::Vector3d that contains the lengths of the three axes:
```cpp
shape = std::make_shared<EllipsoidShape>(
      default_shape_height*Eigen::Vector3d::Ones());
```
Since we actually want a sphere, all three axis lengths will be equal, so we can create an `Eigen::Vector3d` filled with ones by using Eigen::Vector3d::Ones() and then multiply it by the length that we actually want for the three components.

Finally, we want to add this shape as a visualization and collision shape for the BodyNode:
```cpp
  bn->addVisualizationShape(shape);
  bn->addCollisionShape(shape);
```
We want to do this no matter which type was selected, so those two lines of code should be after all the condition statements.

### Lesson 1d Set up the physical properties for the body
For the simulations to be physically accurate, it’s important for the inertia properties of the body to match up with the geometric properties of the shape. We can create an `Inertia` object and set its values based on the shape’s geometry, then give that `Inertia` to the BodyNode.
```cpp
  Inertia inertia;
  double mass = default_shape_density * shape->getVolume();
  inertia.setMass(mass);
  inertia.setMoment(shape->computeInertia(mass));
  bn->setInertia(inertia);
```
For the collision behaviour of we need to set a coefficient of restitution.
This is easily done with:
```cpp
  bn->setRestitutionCoeff(default_restitution);
```
In real life, joints have friction. This pulls energy out of systems over time, and makes those systems more stable. In our simulation, we’ll ignore air friction, but we’ll add friction in the joints between bodies in order to have better numerical and dynamic stability:
```cpp
    if(parent)
    {
      Joint* joint = bn->getParentJoint();
      for(size_t i=0; i < joint->getNumDofs(); ++i)
        joint->getDof(i)->setDampingCoefficient(default_damping_coefficient);
    }
```
If this BodyNode has a parent BodyNode, then we set damping coefficients of its Joint to a default value.

## Lesson 2 Setting initial conditions and taking advantages of Frames
Find the `addObject` function in the `MyWorld` class. This function will be called whenever the user requests for an object to be added to the world. In this function, we want to set up the initial conditions for the object so that it gets thrown at the wall. We also want to make sure that it’s not in collision with anything at the time that it’s added, because that would result in problems for the simulation.

### Lesson 2a Set the starting position for the object
We want to position the object in a reasonable place for us to throw it at the wall. We also want to have the ability to randomize its location along the y-axis.

First, let’s create a zero vector for the position:
```cpp
    Eigen::Vector6d positions(Eigen::Vector6d::Zero());
```
You’ll notice that this is an Eigen::Vector6d rather than the usual Eigen::Vector3d. This vector has six components because the root BodyNode has 6 degrees of freedom: three for orientation and three for translation. Because we follow Roy Featherstone’s Spatial Vector convention, the first three components are for orientation using a logmap (also known as angle-axis) and the last three components are for translation.

First, if randomness is turned on, we’ll set the y-translation to a randomized value:
```cpp
    if(mRandomize)
    positions[4] = default_spawn_range * mDistribution(mMT);
```

`mDistribution(mMT)` will generate a random value in the range [-1, 1] inclusive because of how we initialized the classes in the constructor of `MyWindow`.

Then we always set the height to the default value:
```cpp
    positions[5] = default_start_height;
```
Finally, we use this vector to set the positions of the root Joint:
```cpp
    object->getJoint(0)->setPositions(positions);
```
We trust that the root Joint is a FreeJoint with 6 degrees of freedom because of how we constructed all the objects that are going to be thrown at the wall: They were all given a FreeJoint between the world and the root BodyNode.

Every object in the world is required to have a non-empty unique name. Just like Joint names in a Skeleton, if we pass a Skeleton into a world with a non-unique name, the world will print out a complaint to us and then rename it. So avoid the ugly printout, we’ll make sure the new object has a unique name ahead of time:
```cpp
    object->setName(object->getName()+std::to_string(mSkelCount++));
```
### Lesson 2b Creating reference frames
DART has a unique feature that we call Frame Semantics. The Frame Semantics of DART allow you to create reference frames and use them to get and set data relative to arbitrary frames. There are two crucial Frame types currently used in DART: `BodyNodes` and `SimpleFrames`.

The BodyNode class does not allow you to explicitly set its transform, velocity, or acceleration properties, because those are all strictly functions of the degrees of freedom that the BodyNode depends on. Because of this, the BodyNode is not a very convenient class if you want to create an arbitrary frame of reference. Instead, DART offers the `SimpleFrame` class which gives you the freedom of arbitarily attaching it to any parent Frame and setting its transform, velocity, and acceleration to whatever you’d like. This makes SimpleFrame useful for specifying arbitrary reference frames.

We’re going to set up a couple SimpleFrames and use them to easily specify the velocity properties that we want the Skeleton to have. First, we’ll place a SimpleFrame at the Skeleton’s center of mass:
```cpp
    Eigen::Isometry3d centerTf(Eigen::Isometry3d::Identity());
    centerTf.translation() = object->getCOM();
    SimpleFrame center(Frame::World(), "center", centerTf);
```
Calling `object->getCOM()` will tell us the center of mass location with respect to the World Frame. We use that to set the translation of the SimpleFrame’s relative transform so that the origin of the SimpleFrame will be located at the object’s center of mass.

Now we’ll set what we want the object’s angular and linear speeds to be:
```cpp
    double angle = default_launch_angle;
    double speed = default_start_v;
    double angular_speed = default_start_w;
    if(mRandomize)
    {
      angle = (mDistribution(mMT) + 1.0)/2.0 *
          (maximum_launch_angle - minimum_launch_angle) + minimum_launch_angle;

      speed = (mDistribution(mMT) + 1.0)/2.0 *
          (maximum_start_v - minimum_start_v) + minimum_start_v;

      angular_speed = mDistribution(mMT) * maximum_start_w;
    }
```
We just use the default values unless randomization is turned on.

Now we’ll convert those speeds into directional velocities:
```cpp
  Eigen::Vector3d v = speed * Eigen::Vector3d(cos(angle), 0.0, sin(angle));
  Eigen::Vector3d w = angular_speed * Eigen::Vector3d::UnitY();
```
And now we’ll use those vectors to set the velocity properties of the SimpleFrame:
```cpp
  center.setClassicDerivatives(v, w);
```
The `SimpleFrame::setClassicDerivatives()` allows you to set the classic linear and angular velocities and accelerations of a SimpleFrame with respect to its parent Frame, which in this case is the World Frame. In DART, classic velocity and acceleration vectors are explicitly differentiated from spatial velocity and acceleration vectors. If you are unfamiliar with the term “spatial vector”, then you’ll most likely want to work in terms of classic velocity and acceleration.

Now we want to create a new SimpleFrame that will be a child of the previous one:
```cpp
  SimpleFrame ref(&center, "root_reference");
```
And we want the origin of this new Frame to line up with the root BodyNode of our object:
```cpp
  ref.setRelativeTransform(object->getBodyNode(0)->getTransform(&center));
```
Now we’ll use this reference frame to set the velocity of the root BodyNode. By setting the velocity of the root BodyNode equal to the velocity of this reference frame, we will ensure that the overall velocity of Skeleton’s center of mass is equal to the velocity of the `center` Frame from earlier.
```cpp
  object->getJoint(0)->setVelocities(ref.getSpatialVelocity());
```
Note that the FreeJoint uses spatial velocity and spatial acceleration for its degrees of freedom.

Now we’re ready to toss around objects!

## Lesson 3 Setting joint spring and damping properties
Find the `setupRing` function. This is where we’ll setup a chain of BodyNodes so that it behaves more like a closed ring.

We’ll want to set the stiffness and damping coefficients of only the DegreesOfFreedom that are between two consecutive BodyNodes. The first six degrees of freedom are between the root BodyNode and the World, so we don’t want to change the stiffness of them, or else the object will hover unnaturally in the air. But all the rest of the degrees of freedom should be set:
```cpp
  for(size_t i=6; i < ring->getNumDofs(); ++i)
  {
    DegreeOfFreedom* dof = ring->getDof(i);
    dof->setSpringStiffness(ring_spring_stiffness);
    dof->setDampingCoefficient(ring_damping_coefficient);
  }
```
To make sure that the ring’s rest position works well for the structure it has. Using basic geometry, we know we can compute the exterior angle on each edge of a polygon like so:
```cpp
  size_t numEdges = ring->getNumBodyNodes();
  double angle = 2*M_PI/numEdges;
```
Now it’s important to remember that the joints we have between the BodyNodes are BallJoints, which use logmaps (a.k.a. angle-axis) to represent their positions. The BallJoint class provides a convenience function for converting rotations into a position vector for a BallJoint. A similar function also exists for EulerJoint and FreeJoint.
```cpp
  for(size_t i=1; i < ring->getNumJoints(); ++i)
  {
    Joint* joint = ring->getJoint(i);
    Eigen::AngleAxisd rotation(angle, Eigen::Vector3d(0, 1, 0));
    Eigen::Vector3d restPos = BallJoint::convertToPositions(
          Eigen::Matrix3d(rotation));
  }
```
Now we can set the rest positions component-wise:
```cpp
    for(size_t j=0; j<3; ++j)
    joint->setRestPosition(j, restPos[j]);
```
Finally, we should set the ring so that all the degrees of freedom (past the root BodyNode) start out in their rest positions:
```cpp
  for(size_t i=6; i < ring->getNumDofs(); ++i)
  {
    DegreeOfFreedom* dof = ring->getDof(i);
    dof->setPosition(dof->getRestPosition());
  }
```
And that’s it! You’re now ready to run the program!