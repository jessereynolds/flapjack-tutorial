# ![Flapjack](https://raw.github.com/flpjck/flapjack/gh-pages/images/flapjack-2013-notext-transparent-50-50.png "Flapjack") flapjack-tutorial

A tutorial on getting started with Flapjack http://flapjack.io/

## Overview

In this tutorial, you'll get Flapjack running in a VM using Vagrant and VirtualBox. You'll learn how to:

- Configure Flapjack
- Simulate a failure of a monitored service
- Integrate flapjack with nagios (or icinga) so flapjack takes over alerting
- Import contacts and entities
- Configure notification rules, intervals, and summary thresholds

## Getting flapjack up and running with vagrant-flapjack

Optionally, import the Vagrant provided precise64 box. You can [download it first](http://files.vagrantup.com/precise64.box) and then import it:

```bash
vagrant box add precise64 precise64.box
```

Or download and add in one step:

```bash
vagrant box add precise64 http://files.vagrantup.com/precise64.box
```

Even if you haven't pre-added the precise64 box, the following will also download and install it the first time you run `vagrant up`

Do the thing:

```
git clone https://github.com/flpjck/vagrant-flapjack.git
cd vagrant-flapjack
vagrant up
```

More information: [vagrant-flapjack](https://github.com/flpjck/vagrant-flapjack)

### Verify:

Visit [http://localhost:3080](http://localhost:3080) from your host workstation and you should see the flapjack web UI.

You should also find Icinga and Nagios UIs running at:

- [localhost:3083/nagios3](http://localhost:3083/nagios3) u: `nagiosadmin`, p: `nagios`
- [localhost:3083/icinga](http://localhost:3083/icinga) u: `icingaadmin`, p: `icinga`

## Get comfortable with the Flapjack CLI

SSH into the VM:

```bash
vagrant ssh
```

Have a look at the commands:

```bash
export PATH=$PATH:/opt/flapjack/bin
# ...
flapjack --help
# ...
simulate-failed-check --help
# ...
```

Have a look at the executables under `/opt/flapjack/bin`. Details of these are also available [on the wiki](https://github.com/flpjck/flapjack/wiki/USING#running).

[TODO: get all the binscripts in the PATH of the vagrant user by default]

## Simulate a check failure

Run something like:

```bash
simulate-failed-check fail-and-recover \
  --entity foo-app-01.example.com \
  --check Sausage \
  --time 1
```

### Verify:

Reload the flapjack web UI and you should now be able to see the status of the check you're simulating, e.g. at:

  [http://localhost:3080/check?entity=foo-app-01.example.com&check=Sausage](http://localhost:3080/check?entity=foo-app-01.example.com&check=Sausage)

## Integrate Flapjack with Nagios (and Icinga)

Both Nagios and Icinga are configured already to append check output data to the following named pipe: `/var/cache/icinga/event_stream.fifo`.

All that remains is to configure `flapjack-nagios-receiver` to read from this named pipe, configure its Redis connection, and start it up.

- Edit the Flapjack config file at `/etc/flapjack/flapjack_config.yaml`
- Find the `nagios-receiver` section under `production` and change the `fifo: ` to be `/var/cache/icinga/event_stream.fifo`
- Start it up with:

```
sudo /etc/init.d/flapjack-nagios-receiver start
```

More details on configuration are avilable [on the wiki](https://github.com/flpjck/flapjack/wiki/USING#configuring-components).

### Verify:

Reload the Flapjack web interface and you should now see the checks from Icinga and/or Nagios appearing there.

## Create contacts Ada and Charles

`curl ...`

[TODO: include an example contacts json file in vagrant-flapjack as a starting point]

## Create entities foo-app-01 and foo-db-01 (.example.com)

We're going to assign both Ada and Charles to foo-app-01, and just Ada to foo-db-01.

`curl ...`

[TODO: include an example entities json file in vagrant-flapjack to a starting point]

## Add some notification rules

Charles wants to receive critical alerts by both SMS and email, and warnings by email only. Charles never wants to see an unknown alert.

`curl ...`

Test with:

`simulate-failed-check ...`

----

Charles looks after disk utilisation problems and he's not very good at it, so Ada wants to never receive warnings about disk utilisation checks, but she still wants to be notified when they go critical.

`curl ...`

Test with:

`simulate-failed-check ...`

### Your turn

Ada wants to receive critical and warning alerts by both SMS and email, and wants to receive unknown alerts by email only.

- Come up with the JSON import actions to achieve this.
- Create and run the `simulate-failed-check ...` commands to test you've set the contact data up correctly.

## Modify the notification intervals

Ada wants to be notified at most every 3 hours by email, and every 5 minutes by sms:


### Your turn

Charles wants to be notified at most every 30 minute by email, and every 10 minutes by sms.

## Modify summary thresholds

Ada wants to have email alerts rolled up when there's 5 or more alerting checks, and to have sms alerts rolled up no matter how many alerting checks she has.

### Your turn

Charles wants to have email alerts rolled up when there's 10 or more alerting checks, and to have sms alerts rolled up when there's 3 or more alerting checks.

# FIN

===========

# TODO:

- add instructions for using a alternative Vagrant providers, eg AWS, VMWare Fusion
- add a jabber daemon with MUC, eg ejabberd, and include steps to integrate and test

## Start up the fake SMTP server (mailcatcher)

[FIXME: system Ruby is a bit borked on the VM at the moment and installing `mailcatcher` gem is failing]

`mailcatcher`

If it's not installed, install it like this:

`sudo gem install mailcatcher`

Open up [http://localhost:1025](http://localhost:1025) - this will update dynamically whenever it catches an email generated by flapjack.

[TODO: add mailcatcher to vagrant-flapjack if it is not already, and forward the port for it]
