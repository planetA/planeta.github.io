---
title: What are bots?
date: 2018-01-02 13:38
categories: Social
tags: ccc review information-security
---

Recently I watched a presentation from the last CCC called ["Social
Bots, Fake News und
Filterblasen"](https://media.ccc.de/v/34c3-9268-social_bots_fake_news_und_filterblasen). The
presentation was give by a *data journalist* [Michael
Kreil](https://media.ccc.de/search?q=Michael+Kreil), who applies data
science toolkit to social networks. Mostly he gets data from Twitter,
because Twitter public API enables comparatively easy access to user
publications and user records. He analyses the data he can gather and
publishes reports about, for example, social profile of a typical
person spreading fake news stories.

When doing such a research it is natural to question if the accounts
under study backed by real people? One particular phenomenon usually
contrasted to real people often called *social bot*. Surprisingly
enough the question who is a bot and who is a person is not so easy to
answer, because on the Internet you don't another side of the
cable. What is social bot and how to identify it was one of the main
topics in the talk. Of course the discussion started with the very
first question about social bots: How one would define a social bot?
Among all possible definitions the presenter also was interested in a
scientifically unambiguous one. And as turned out the more scientific
the definition is, the harder is to count the bots?  I agree that the
question of counting is important, nevertheless I think that the way
how Michael defined social bots missed or at least understated the
root: what is our motivation to study social bots?

In the talk we see the definition of a social bot given by <abbr
title="Büro für Technikfolgen-Abschätzung beim Deutschen
Bundestag">TAB</abbr> in a position paper [Social
Bots](https://www.bundestag.de/blob/488564/4a87d2d5b867b0464ef457831fb8e642/thesenpapier-data.pdf)(my
translation):

> Social bot is a computer program, created with a manipulative
> purpose, which imitates a human personality and communicates with
> people on the Internet.

Once there is a definition one requires a method to count entities who
fall under. This question is hard to answer, because a successful
social bot can pass a Turing test and thus is indistinguishable from a
human. The presenter discusses various methods (around 30:00+) to
figure out who is a bot on Twitter. The methods range from primitive
ones, like having a threshold for number of tweets per day, to
sophisticated machine learning based algorithms which analyze the
whole network of twitter-followers of a given account. The problem
with the former is being scientifically silly. The problem with
the later is being hardly verifiable.

From my point of view any possible way to identify social bots in a
sense of aforementioned definition is conceptually problematic. A
simple method knowingly ignores data exposing computer nature of an
account. A sophisticated one has to play a Turing test game better
than humans. This is hard to do, and still an ideal social bot is
truly indistinguishable from an individual. Michael Kreil acknowledges
that it is possible to estimate if an account run by a program only
with a certain confidence.

I think the whole issue of counting bots comes from this precise
definition of what a bot is. Breaking the definition into individual
statements we see that a social bot is a computer program. It is
deemed to manipulate human opinion. And a social bot communicates with
other people. It is not too far fetched to say that any Twitter client
matches the definition. Any Twitter client is a computer program. A
Twitter client manipulates people opinion on behalf of the user of the
client. And of course a twitter client is supposed to be reachable by
other people. All in all social bots also act on behalf of real
people. Then what is the difference between a social bot from TAB
definition and a usual Twitter client which publishes posts on behalf
of its user? Not so much!

The reason we care about social bots not because we are afraid that
there is a too thick software layer between people, but because when
people exchange opinions, they want to know what does the society
around them thinks. Most people are hardwired to be susceptible to
judgements of other people. Since social bots pretend to represent an
opinion of some real person, in the worst case they can trick more
susceptible individuals in believing in a fiction. We much easier
believe that something terribly unjust happens around us of our circle
of reference says that. When such belief is established, anybody can
become radicalized, even if the reality gives no apparent reason for
such kind of belief.

Those who are not tricked into false narratives of social bots are
also affected. Seeing it as an attempt of a deception, post of social
bots easily become a source of anger. Still people are equally angry
if they find themselves trying to dialogue with a real person, but
whose sole purpose is to provoke them. The whole anxiety about social
bots is very similar in nature to distress about [internet
trolls](https://en.wikipedia.org/wiki/Internet_troll). We don't care
how much compute power was put into making something, what we can't
consider to be a valid opinion. Thus the definition of a social bot as
a computer program superfluous for most practical purposes.

This intuition behind the term is coherent with frequently used
colloquial understanding social bots as a concept. Infamous [Internet
Research
Agency](https://en.wikipedia.org/wiki/Internet_Research_Agency)
operates hiring real people, who post opinions under fake identities
with an attempt to create an impression that certain opinions are
represented more than they actually are
([one](https://www.novayagazeta.ru/articles/2013/09/07/56253-gde-zhivut-trolli-i-kto-ih-kormit),
[two](https://www.theguardian.com/world/2015/apr/02/putin-kremlin-inside-russian-troll-house)).
Employees of IRA are often called *trolls from Olgino* (by the office
location) or *kremlebots* (by their political orientation). Note, how
words troll and bot are used interchangeably. As the first article
describes the kremlbots have to post at least 50 comments per day on
social networks. From the first glance, this would qualify them as
social bots according to criterion mentioned by Michael Kreil. But the
trolls have to maintain at least 6 Facebook accounts, which means that
on average one account will publish less than 10 posts. Such a rate is
even less than some moderate social network users. And still this
group of people should be included into the definition of social bot,
otherwise the definition simply omits an important dimension of the
phenomenon it is deemed to describe. As an alternative, I propose
following definition of a social bot.

**Social bot** is a social media account which represents *non-unique
identity of a human person*. **Unique identity** of a human person is an
online identity, which uniquely maps to a human person, represents a
specific set of interests, thoughts and opinions of this person and
does not covertly overlap with other active unique identities of the
same person.

A unique identity is a new concept in the definition, which is usually
an account on a social network used for a particular purpose. For
example, a user may have two Twitter profiles, one used to publish
information related to professional activity, another one used to
publish information about personal life. Even if these two accounts
are not explicitly connected few people will be deceived into thinking
that the two accounts represent two opinions.

Even if it happens so, that, for example, the first account retweets a
post of the second as long the fact that the two accounts are related
to the same person remains clear, nobody will be deceived into
perceiving original post and retweet as opinions of separate people.
This definition does not require accounts to be explicit about who
they are mapped to. Both accounts can stay pseudonymous. Even if the
user creates a new account for each and every new post, neither of the
accounts falls under the definition of social bot, because if an old
account is discontinued, it can't overlap with a newly created
account. Of course, to avoid confusion, the fact that the account is
discontinued should be evident.

Another example is a [CNN tweeter bot](https://twitter.com/CNN)
posting definitely more than 50 tweets per day, but still not
qualifying for a social bot, because it unambiguously represents an
organisation, not a person. On the other hand an IRA employee, even if
posting less than 50 comments per day, qualifies as a social bot,
because the accounts the employee runs map to the same human and
express opinions on related topics.

This definition works around an unsolvable problem of identifying how
much of algorithmic effort was put into creating a post on social
media. On the other hand this suggests that analysis on how many
social bots are there on the network purely from the account behavior
renders to be infeasible. We're definitely past the time when such
analysis could be considered reliable, so better not even try.

Such a conclusion does not mean that you can't trust anybody on the
Internet. I just think that to factor out social bots one should do it
on a technical level. A crude example of technical measures for human
validation is provided by Tweeter and Facebook. These both companies
have a teams of specialists whose task is to check an account holder
credentials and prove that the displayed name matches the person's
real name. After a successful verification an account gets a special
sign signalizing others that the account holder was validated. I see a
number of shortcomings of this process. First, it is quite cumbersome
and expensive. Few companies provide it, and still only for small
fraction of users. Second, it restricts the user anonymity.

I believe better measures are possible. A properly built technical
system should have much less personnel interaction for issuing new
user identities and still allow more freedom for anonymity. If a
social network consists only of users who pass this form of
verification the problem of social bots simply falls off.
