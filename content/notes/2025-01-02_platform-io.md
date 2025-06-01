+++
title = "PlatformIO"

[taxonomies]
tags = [ "embedded", "pio", "setup" ]
+++


By default, PlatformIO creates new projects inside your Documents directory, which might not be ideal for everyone. If you prefer to keep your projects elsewhere, like me, you can easily change the default location in the PlatformIO terminal with the following command:

```sh
pio settings set projects_dir  ~/Development/platformIO
```

Just replace `~/Development/platformIO` with your preferred folder.
This way, all new projects will be created in the directory you choose, keeping your workspace organized the way you like it.
