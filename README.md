I’m running into an environment issue and was hoping you could help. I’ve tried using keras==2.6.0 to match the legacy code that imports Adam from keras.optimizers, but that results in an import error for Adam. However, if I upgrade to a newer version of Keras (e.g., 2.11.0) to resolve that, I then get another error:

pgsql
Copy
Edit
ImportError: cannot import name 'dtensor' from 'tensorflow.compat.v2.experimental'
It seems like the Keras and TensorFlow versions in the current environment are mismatched — one is too old, and the other is too new — and I can’t modify the library code to adjust the import paths.

I’ve attached screenshots of both errors. Could you please advise if there’s a supported environment setup that would resolve this?

Thanks so much,
Tianyi
