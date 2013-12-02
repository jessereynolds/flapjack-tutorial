# ![Flapjack](https://raw.github.com/flpjck/flapjack/gh-pages/images/flapjack-2013-notext-transparent-50-50.png "Flapjack") Flapjack Tutorial

A tutorial on getting started with [Flapjack](http://flapjack.io/).

## Overview

In this tutorial, you'll get Flapjack running in a VM using Vagrant and VirtualBox. You'll learn how to:

- Configure Flapjack
- Simulate a failure of a monitored service
- Integrate flapjack with nagios (or icinga) so flapjack takes over alerting
- Import contacts and entities
- Configure notification rules, intervals, and summary thresholds

## Getting flapjack up and running with vagrant-flapjack

### Prerequites

- Vagrant 1.2+
- VirtualBox 4.2+ (or an alternative provider such as VMWare Fusion)
- precise64.box added to your vagrant (optional pre step)

### Adding the precise64.box vagrant box (VirtualBox specific, optional)

In order to speed up the `vagrant up` step below, optionally import the Vagrant provided precise64 box. You can [download it first](http://files.vagrantup.com/precise64.box) and then import it:

``` shell
vagrant box add precise64 precise64.box
```

Or download and add in one step:

``` shell
vagrant box add precise64 http://files.vagrantup.com/precise64.box
```

Install the `vagrant-cachier` plugin for Vagrant to speed up subsequent `vagrant up`'s:

``` shell
vagrant plugin install vagrant-cachier
```

Even if you haven't pre-added the precise64 box, the following will also download and install it the first time you run `vagrant up`

Do the thing:

``` shell
git clone https://github.com/flpjck/vagrant-flapjack.git
cd vagrant-flapjack
vagrant up
```

For an alternative provider to VirtualBox, eg VMWare Fusion, you can specify the alternative provider when running `vagrant up` like so:

``` shell
vagrant up --provider=vmware_fusion
```

More information: [vagrant-flapjack](https://github.com/flpjck/vagrant-flapjack)

### Verify:

Visit [http://localhost:3080](http://localhost:3080) from your host workstation and you should see the flapjack web UI.

You should also find Icinga and Nagios UIs running at:

- [localhost:3083/nagios3](http://localhost:3083/nagios3) u: `nagiosadmin`, p: `nagios`
- [localhost:3083/icinga](http://localhost:3083/icinga) u: `icingaadmin`, p: `icinga`

## Get comfortable with the Flapjack CLI

SSH into the VM:

``` shell
vagrant ssh
```

Have a look at the executables under `/opt/flapjack/bin`. Details of these are also available [on the wiki](https://github.com/flpjck/flapjack/wiki/USING#running).

``` shell
export PATH=$PATH:/opt/flapjack/bin
# ...
flapjack --help
# ...
simulate-failed-check --help
# ...
```

TODO: get all the binscripts in the PATH of the vagrant user by default [omnibus-flapjack#11](https://github.com/flpjck/omnibus-flapjack/issues/11)

## Simulate a check failure

Run something like:

``` shell
simulate-failed-check fail-and-recover \
  --entity foo-app-01.example.com \
  --check Sausage \
  --time 3
```

This will send a stream of critical events for 3 minutes and send one ok event at the end. If you want the last event to be a failure as well, use the `fail` command instead of `fail-and-recover`

### Verify:

Reload the flapjack web UI and you should now be able to see the status of the check you're simulating, e.g. at:

[http://localhost:3080/check?entity=foo-app-01.example.com&check=Sausage](http://localhost:3080/check?entity=foo-app-01.example.com&check=Sausage)

## Integrate Flapjack with Nagios (and Icinga)

Both Nagios and Icinga are configured already to append check output data to the following named pipe: `/var/cache/icinga/event_stream.fifo`.

All that remains is to configure `flapjack-nagios-receiver` to read from this named pipe, configure its Redis connection, and start it up.

- Edit the Flapjack config file at `/etc/flapjack/flapjack_config.yaml`
- Find the `nagios-receiver` section under `production` and change the `fifo: ` to be `/var/cache/icinga/event_stream.fifo`
- Start it up with:

``` shell
sudo /etc/init.d/flapjack-nagios-receiver start
```

More details on configuration are avilable [on the wiki](https://github.com/flpjck/flapjack/wiki/USING#configuring-components).

### Verify:

Reload the Flapjack web interface and you should now see the checks from Icinga and/or Nagios appearing there.

## Create some contacts and Entities
 
Currently Flapjack does not include a friendly web interface for managing contacts and entities, so for now we use json, curl, and the [Flapjack API](https://github.com/flpjck/flapjack/wiki/API). 

The vagrant-flapjack project ships with [example json files](https://github.com/flpjck/vagrant-flapjack/tree/master/examples) that you can use in this tutorial, or you can copy and paste the longform curl commands below that include the json. 

### Create Contacts Ada and Charles
 
We'll be using the [POST /contacts](https://github.com/flpjck/flapjack/wiki/API#wiki-post_contacts) API call to create two contacts.

Run the following from your workstation, cd'd into the vagrant-flapjack directory:

``` shell
curl -w 'response: %{http_code} \n' -X POST -H "Content-type: application/json" \
  -d @examples/contacts_ada_and_charles.json \
  http://localhost:3081/contacts
```

Or alternatively, copy and paste the following. This does the same thing, but includes the json data inline. 

``` shell
curl -w 'response: %{http_code} \n' -X POST -H "Content-type: application/json" -d \
 '{
    "contacts": [
      {
        "id": "21",
        "first_name": "Ada",
        "last_name": "Lovelace",
        "email": "ada@example.com",
        "media": {
          "sms": {
            "address": "+61412345678",
            "interval": "3600",
            "rollup_threshold": "5"
          },
          "email": {
            "address": "ada@example.com",
            "interval": "7200",
            "rollup_threshold": null
          }
        },
        "tags": [
          "legend",
          "first computer programmer"
        ]
      },
      {
        "id": "22",
        "first_name": "Charles",
        "last_name": "Babbage",
        "email": "charles@example.com",
        "media": {
          "sms": {
            "address": "+61412345679",
            "interval": "3600",
            "rollup_threshold": "5"
          },
          "email": {
            "address": "charles@example.com",
            "interval": "7200",
            "rollup_threshold": null
          }
        },
        "tags": [
          "legend",
          "polymath"
        ]
      }
    ]
  }' \
 http://localhost:3081/contacts
```


### Create entities foo-app-01 and foo-db-01 (.example.com)

We'll be using the [POST /entities](https://github.com/flpjck/flapjack/wiki/API#wiki-post_entities) api call to create two entities. 

We're going to assign both Ada and Charles to foo-app-01, and just Ada to foo-db-01.

``` shell
curl -w 'response: %{http_code} \n' -X POST -H "Content-type: application/json" \
  -d @examples/entities_foo-app-01_and_foo-db-01.json \
  http://localhost:3081/entities
```

Or with json inline if you prefer:

``` shell
curl -w 'response: %{http_code} \n' -X POST -H "Content-type: application/json" -d \
 '{
    "entities": [
      {
        "id": "801",
        "name": "foo-app-01.example.com",
        "contacts": [
          "21",
          "22"
        ],
        "tags": [
          "foo",
          "app"
        ]
      },
      {
        "id": "802",
        "name": "foo-app-02.example.com",
        "contacts": [
          "21"
        ],
        "tags": [
          "foo",
          "db"
        ]
      }

    ]
  }' \
 http://localhost:3081/entities
```

### Add some notification rules

Charles wants to receive critical alerts by both SMS and email, and warnings by email only. Charles never wants to see an unknown alert.

`curl ...`

Test with:

`simulate-failed-check ...`

----

Charles looks after disk utilisation problems and he's not very good at it, so Ada wants to never receive warnings about disk utilisation checks. She still wants to be notified when they go critical.

`curl ...`

Test with:

`simulate-failed-check ...`

#### Your turn

Ada wants to receive critical and warning alerts by both SMS and email, and wants to receive unknown alerts by email only.

- Come up with the JSON import actions to achieve this.
- Create and run the `simulate-failed-check ...` commands to test you've set the contact data up correctly.

### Modify the notification intervals

Ada wants to be notified at most every 3 hours by email, and every 5 minutes by sms:


#### Your turn

Charles wants to be notified at most every 30 minute by email, and every 10 minutes by sms.

### Modify summary thresholds

Ada wants to have email alerts rolled up when there's 5 or more alerting checks, and to have sms alerts rolled up no matter how many alerting checks she has.

#### Your turn

Charles wants to have email alerts rolled up when there's 10 or more alerting checks, and to have sms alerts rolled up when there's 3 or more alerting checks.

## Feedback

How did you go with this tutorial? If you have feedback on how it can be improved, or just to say "this is awesome" (or whatever), please put it out there by doing something like:

- creating an [issue](https://github.com/jessereynolds/flapjack-tutorial/issues)
- submitting a [pull request](https://github.com/jessereynolds/flapjack-tutorial/pulls)
- jumping on [irc](irc:///#flapjack) (#flapjack on freenode)
- writing to the [mailing list](https://groups.google.com/forum/#!forum/flapjack-project)
- tweeting @jessereynolds or @auxesis 

# ~ FIN ~

===========

# TODO:

- add instructions for using alternative Vagrant providers, eg AWS, VMWare Fusion
- add a jabber daemon with MUC, eg ejabberd, and include steps to integrate and test

## Start up the fake SMTP server (mailcatcher)

[FIXME: system Ruby is a bit borked on the VM at the moment and installing `mailcatcher` gem is failing]

`mailcatcher`

If it's not installed, install it like this:

`sudo gem install mailcatcher`

Open up [http://localhost:1025](http://localhost:1025) - this will update dynamically whenever it catches an email generated by flapjack.

[TODO: add mailcatcher to vagrant-flapjack if it is not already, and forward the port for it]
