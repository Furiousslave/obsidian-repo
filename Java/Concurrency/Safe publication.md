Safe publication provides that threads can see the object in it's aspublished state.
To publish an object safely, both the referesence to the object and the object's state must be made visible to other threads at the same time. A properly constructed object can be safely published by:
- Initializing an object reference from a static initializer
- Storing a reference to it into volatile field or AtomicReference
- Storing reference to in into a final field of a properly constructed object
- Storing a reference to in into a field that is properly guarded by a lock