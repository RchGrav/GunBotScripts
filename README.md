# GunBot config management scripts

This is not a replacement for understanding how to configure gunbot, but instead is meant to illustrate one possible deployment scenario utilizing multiple folders..  - check your own configs carefully if using these scripts, and be aware that you are fully responsible for whatever your GunBot does.   I appreciate your feedback.  This project is forked from another github user because I liked what he did with his but this one is being rewritten piece by piece.  As it evolves it may look less and less like the original project.  The name is the same to protect the innocent. Although it may need to be changed at some point as it mutates away from the original code..  We shall see... 

![alt text](http://i.imgur.com/sMnFZro.png)

## Getting Started

The scripts assume gunbot is in /opt/gunbot (or there's a simlink there to it's actual location). All that is needed in /opt/gunbot prior to running
the scripts in gunthy-linuxx64 chmodded +x and you're ready to go. The instructions assume a recent Ubuntu, for other Linux you'll need to adapt them.

If you don't want Gunbot or a symlink to it in /opt/gunbot you'll need to edit the GUNBOTROOT variable in the scripts

To get started, as root:

```
cd /opt/gunbot; sudo apt-get install git; git clone https://github.com/RchGrav/GunBotScripts; mv GunBotScripts/scripts/ .; rm -rf GunBotScripts 
```

## Configuration

Inside /opt/gunbot you will find 2 folders (templates and pairs) with files inside them for you to configure.

## Per-exchange template files

In the templates folder are 3 template files, one for each exchange (Poloniex, Bittrex, Kraken). If you're not using one or more of the exchanges, simply leave
it's pairs file empty and you can leave it's template in place for future use at it will simply be ignored.

Each template file contains settings that it seemed to me sensible to set per-exchange such as trade size, API timings, and trading strategy. Since those
templates will create individual config files like poloniex-BTC_ALT-config.js when the scripts are run, those can then be individually edited later if
for example you wish to employ different strategies per pair.

Trend follower sensitivity is also set in the per-exchange template files via MAX_LATEST_PRICES_TREND, this variable determines how many prices are
being used by the trend follower, so since timings are different per exchange (it's possible to poll Bittrex for example much faster than Poloniex)
it also makes sense to adapt this value - if for example, Poloniex is being polled approximately once per minute then using 60 prices for the trend
follower equates to an hour of data. If we were polling Bittrex every 20 seconds, then 60 prices would be only 20 minutes of data, leading to a much
more "reactive" trend-follower that may determine a trend direction too early with insufficient data, therefore we would set a higher MAX_LATEST_PRICES_TREND
for Bittrex.

The default Poloniex timings of approximately (they are slightly randomised) 60s BOT_SLEEP_DELAY and 30s BOT_ON_FAIL_DELAY are good for up to around 10
pairs on Poloniex, if you are running more pairs than that you should adjust them up accordingly, 120s and 60s should suffice for up to 20 pairs, if you
set your timings on Poloniex too fast you will get more 422 errors and possibly get your API key banned.


## Global ALLPAIRS-params.js

The ALLPAIRS-params.js in the templates folder is copied over the existing file in /opt/gunbot by the scripts. It contains global settings
that it made sense to me to have set centrally. Be aware that in GunBot, ALLPAIRS has priority - that is to say, if a setting is in BOTH the individual
config.js file for the pair (generated from the per-exchange templates) and ALSO in the ALLPAIRS file, the setting in the ALLPAIRS file will be the
one that applies. For that reason, the only setting that is in the ALLPAIRS template as supplied which duplicates on in the per-exchange templates
is BTC_TRADING_LIMIT - in the template ALLPAIRS it is set to 1 satoshi and commented out. It therefore provides a convenient way to pause buying - all
you need to do is uncomment it in the /opt/gunbot/ALLPAIRS-params.js and save the file, GunBot will pick up the change whilst running without any
need to restart, and will not buy anything since the trading limit will be below the exchange minimum order size, therefore you will have gunbot
only selling which is useful for example when dealing with bags.

Because of the ALLPAIRS priority, you may wish to move some variables either from ALLPAIRS to the per-exchange templates, or vice-versa, depending
on your preferences, just make sure you don't leave a variable in both unless you are absolutely sure you want the ALLPAIRS to override the per-pair
files.

You will need to edit the ALLPAIRS templates to set, at a minimum, POLONIEX_KEY and POLONIEX_SECRET for your API key and corresponding secret, and
the same values for the other exchanges if you run them.

## Generating the configs and scripts

So you've edited the Pairs files to set up what pairs to trade, you've edited the per-exchange template files to set the size of trades, strategy and
API calls timing if needed, and you've edited the global ALLPAIRS template to include your API keys. 

Take a look inside of the generate-config file and adjust the "Exchange" and "Base" variable to specify which folders you would like to build out.   The BTC base pair will be in the /opt/gunbot folder but all of the rest will follow the naming convention of /opt/gunbot_exchange_pair, based upon which you select.  You are currently limited to the pairs I have configured but it should be self evident how to add more.   

You're now ready to run the generate-configs script. It will generate a bunch of exchange-BTC_PAIR-config.js type files in GUNBOTROOT (normally /opt/gunbot), a bunch of YAML files for PM2
in GUNBOTROOT/YAMLs, a global allpairs.yaml file in GUNBOTROOT, and a bunch of scripts in GUNBOTROOT/scripts such as start-all-pairs

So normally, in the most basic case, at this point, assuming nothing conflicting is already running and it's a fresh start, you could run the
start-all-pairs script which will, with a delay between each, start all the pairs by doing "pm2 start YAMLs/Exchange-BTC_ALT.yaml"

You should now be able to see them all running in pm2 l or using gmon (https://github.com/BeerK0in/gunbot-monitor)

## Adding or removing pairs

No logic exists in the scripts for deciding if a pair is already running. I am not sure, but I suspect wierd things might happen if you ran,
for example, start-all-pairs more than once. Therefore, what I tend to do to add a new pair is edit the file in the Pairs folder, run
generate-configs (which will overwrite all the existing configs in /opt/gunbot which if there's no changes is harmless enough, but may not
be what you want, think it through first if the templates have changed since it was last run, or also if the template ALLPAIRS changed), that
will create a new Exchange-BTC_ALT-config.js in /opt/gunbot for the new coin, and you can now manually start only that pair by cd'ing to
/opt/gunbot and running pm2 start YAMLs/Exchange-BTC_ALT.yaml

To remove a pair, remove it from the Pairs file so it isn't regenerated next time, then just pm2 stop the pair, and manually delete it's
config file and possibly logs, token file etc. if that's what you want.

## Modifying parameter for running bots

Once you have pairs running, you can hot-reconfig them simply by changing the files, either per-pair or ALLPAIRS in /opt/gunbot/
Gunbot will pick up the changes at the next cycle, so if you wish to edit things per pair, or globally, that's the way to do it. 

If you want to change a per exchange-setting such as BTC_TRADING_LIMIT for example then a quicker way to do it for all the pairs on that
exchange is simply to edit the exchange template file then just run generate-configs again to overwrite all the pair configs. Be aware
this process is dumb, and will overwrite your /opt/gunbot/ALLPAIRS-params.js with the template version as well as overwriting all the
pairs on the other exchanges where you didn't change anything in the config, so although it should usually be harmless if those files
haven't changed in /opt/gunbot, it may also not be what you want so think through what it's going to do before running it.

