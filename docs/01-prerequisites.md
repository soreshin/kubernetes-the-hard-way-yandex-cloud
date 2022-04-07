# Prerequisites

## Yandex Cloud Platform

This tutorial is my humble experiment of migration the famous project "Kubernetes The Hard Way" from Google Cloud to Yandex Cloud. You can sign up and get 4000 ₽ (either 24 000 ₸ or 50 $ depends on you location) welcome free credit.

> I think that the compute resources required for this tutorial exceed the Yandex Cloud Platform free tier.


## Yandex Cloud Platform CLI

### Install the Yandex Cloud CLI

Follow the Yandex Cloud CLI [documentation](https://cloud.yandex.com/en-ru/docs/cli/) to install and configure the `yc` command line utility.

Verify the Yandex Cloud CLI version (I tested this guide on Yandex Cloud CLI 0.89.0 version):

```
yc version
```

### Initialize Yandex Cloud CLI

This tutorial assumes a default compute region and zone have been configured.

If you are using the `yc` command-line tool for the first time `init` is the easiest way to do this. You have to receive [OAuth](https://oauth.yandex.ru/authorize?response_type=token&client_id=1a6990aa636648e9b2ef855fa7bec2fb) token to access to Yandex Cloud ([documentation](https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token))

```
yc init
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with synchronize-panes enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
