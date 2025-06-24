Hi [ZAML Team / Team Name],

I’ve been encountering a compatibility issue when trying to run code that imports Adam from keras.optimizers. The current environment appears to include a newer version of TensorFlow (which expects dtensor) but also uses a standalone version of keras that is incompatible with that TensorFlow version.

If I downgrade Keras to 2.6.0 to match the legacy import path, I run into a dtensor import error from TensorFlow. If I try to use tensorflow.keras instead, the code fails because I can't modify the library to update the import paths.

Would it be possible to adjust the environment or provide guidance on how best to proceed? I’d really appreciate your help.

Best regards,
Tianyi
