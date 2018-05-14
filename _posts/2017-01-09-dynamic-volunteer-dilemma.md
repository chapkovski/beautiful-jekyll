---
layout: post
title: Dynamic Volunteer's Dilemma
tags: [python]
categories: [design, oTree]
---

Description of Dynamic Volunteer's Dilemma

I've been recently working on dynamic version of Dieckmann (1985) volunteer's dilemma. The game itself is
an attempt to find a theoretical answer to [the story of Ketty Genovese](https://en.wikipedia.org/wiki/Murder_of_Kitty_Genovese).

The shortest description of the Volunteer's Dilemma (VOD) game can be found in [(Franzen 1996)](#references):

> "Subjects received U = 100 in case of collective good production. Providing the good cost K = 50. Thus a subject who cooperated by choosing alternative A received 100 - 50 = 50 points. If no subject provided the good (.e. all players chose B), each received 0 points."


The game-theoretical predictions are that the cooperation rate iv VOD should decline with increasing group size.

I was wondering if adding a time pressure to the VOD would change the group size effect.

The rules of this new dynamic game are pretty simple:

1. N players have to take a decision who will pay a price.
1. As soon as a single player volunteers to pay the price, everyone in a group receives a certain amount (endowment).
1. The one who agreed to pay, receives this amount minus the price.
1. The price grows every second. As soon as it reaches the endowment, the game ends, and everyone gets 0.

We can look at this game as a sort of inverted [Dutch auction](https://en.wikipedia.org/wiki/Dutch_auction)
(in Dutch auction the price falls until it is accepted by one of the players, or until it reaches the reserve value.)

The technical problem was to make the periodic (every _n_ seconds) price update at the server side. It could be done on the clients' side as well, but then as the number of participants per session grows, the requests to server would become more and more frequent. If we'd like to increase the price smoothly we had to deal with it at the server's side.

The 'right' way of dealing with background tasks in Django (and oTree) would be to use [Celery](http://docs.celeryproject.org/en/latest/) and [Celery Beat](https://github.com/celery/django-celery-beat). Some people believe that Celery is overkill for small tasks. In this case something more lightweighted is used like [Django Background Tasks](https://github.com/arteria/django-background-tasks/).

But both Celery and Django Background Tasks should be run as independent processes, so you can't use free dynos in Heroku
for that, because Heroku gets only 1 free dyno.

The task is trivial: to check whether there are active sessions, and if there are any, increase the price in each group by one  every second. As soon as one of the players click 'Pay the price' the increase of the price is stopped and the payoffs are calculated.

The main code is in `models.py`:

``` python
# we check if the group model exists (otherwise an attempt to retrieve all active
# groups would fail)
def group_model_exists():
    return 'volunteer_group' in connection.introspection.table_names()


class Player(BasePlayer):
    auction_winner = models.BooleanField(initial=False)
    def set_payoff(self):
        self.payoff = (Constants.endowment - self.group.price * self.auction_winner) * (not self.group.timeout)

class Group(BaseGroup):
    price = models.IntegerField(initial=0)
    activated = models.BooleanField()
    timeout = models.BooleanField(initial = False)


def runEverySecond():
  # if the Group model exists we continue
    if group_model_exists():
      # we'll get all the Groups which are activated, i.e.
      # the players arrive to the Auction page there
        activated_groups = Group.objects.filter(activated=True)
        # in all activated groups we increase the price
        for g in activated_groups:
          # if the price is lower than endowment...
            if g.price < Constants.endowment:
                g.price += 1
                g.save()
                # ...and we send the updated price via WebSockets to the specific
                # group members
                channels.Group(
                    'groupid{}'.format(g.pk)
                ).send(
                    {'text': json.dumps(
                        {'price': g.price})}
                )
            else:
              # if the price is above the endomwment we
              # end the game and push the participants to the next page
                g.timeout = True
                g.save()
                advance_participants([p.participant for p in g.get_players()])

# This is the Twisted looping call that is not blocking - it can be run in
# background
l = task.LoopingCall(runEverySecond)
if not l.running:
    l.start(1.0)

```


As usual, the code is [here](https://github.com/chapkovski/every-second-auction) and the [demo app is here](https://guarded-everglades-67747.herokuapp.com/demo/).

## References:

* [Franzen, A., 1995. Group size and one-shot collective action. Rationality and Society, 7(2), pp.183-200.]( http://www.soz.unibe.ch/unibe/portal/fak_wiso/c_dep_sowi/inst_soz/content/e39893/e48983/e127077/e127325/e127456/Franzen-GroupSizeAndOne-ShotCollectiveAction_ger.pdf)
