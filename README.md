###I've since uploaded a better-coded [version](https://github.com/maxantcliff/football_poisson_2/blob/master/football_poisson_2.ipynb).

# Description
My first project - I'd been through the [Jake VanderPlas handbook](https://jakevdp.github.io/PythonDataScienceHandbook/) and was keen to try out an application for numpy and pandas, so this is entirely a learning exercise. It's nothing new, and doesn't generate a reliable ROI, but rather that just copy the code for an advanced model I wanted to build something myself from scratch. Hopefully this helps someone who is also just starting out!

# Resources
You can download each premier league season's dataset [here](https://www.football-data.co.uk/englandm.php). In the code you’ll need to define a season in which to place bets. The results are then predicted based on data from the previous season.

```python
betting_year=17
data_year=betting_year-1
```

# Method
I apply a very basic version of the [Poisson distribution](https://en.wikipedia.org/wiki/Poisson_distribution). The distribution is of the probability of an event occuring a number of times, given its mean incidence:
```python
def poisson(actual, mean):
    return(mean**actual*math.exp(-mean))/math.factorial(actual)
```

The code loads the data for the season immediately before the one you want to make bets on ('data’), makes a list of the teams and calculates the average home and away goals for each team. 
```python
teamlist=[]
for i in range(len(data)):
    if data.HomeTeam[i] not in teamlist:
        teamlist.append(data.HomeTeam[i])
teamlist.sort()
columns=['home_goals', 'away_goals', 'home_conceded', 'away_conceded',
           'home_games', 'away_games','total_games', 'alpha_h','alpha_a']
teaminfo= pd.DataFrame(index=teamlist, columns=columns)
teaminfo[(columns)]=0
for i in range(len(data)):
    teaminfo['home_games'][(data.HomeTeam[i])]+=1
    teaminfo['away_games'][(data.AwayTeam[i])]+=1
    teaminfo['home_goals'][(data.HomeTeam[i])]+=data.FTHG[i]
    teaminfo['away_goals'][(data.AwayTeam[i])]+=data.FTAG[i]
teaminfo['alpha_h']=teaminfo['home_goals']/teaminfo['home_games']
teaminfo['alpha_a']=teaminfo['away_goals']/teaminfo['away_games']
teaminfo['total_games']=teaminfo['home_games']+teaminfo['away_games']
```

Each team's average home and away goals are used as the the expected value in Poisson distributions for the number of goals scored.  For the season you want to make bets on the newly promoted teams' matches are dropped as there isn't data on their average goals in the premier league.
```python
for i in range(len(season)):
    if season.HomeTeam[i] not in teamlist or season.AwayTeam[i] not in teamlist:
        season.promoted[i]=1
season.drop(season[season.promoted==1.0].index,inplace=True)
season.reset_index(drop=True, inplace=True)
```
Then for each match in the edited 'season' dataset, each possible outcome is calculated (up to the maximum goals you set - I set up to 10-10 here) and stored in a temporary dataframe.

```python
maxscore = 11
for game in range(len(season)):    
    probs=pd.DataFrame(index=range(maxscore**2),columns=['homescore','awayscore','probability'])
    index_counter=0
    for i in range(maxscore):
        for j in range(maxscore):
            prob = poisson(i, teaminfo['alpha_h'][season.HomeTeam[game]]) * poisson(j,teaminfo['alpha_a'][season.AwayTeam[game]])
            probs.homescore[index_counter]=i
            probs.awayscore[index_counter]=j               
            probs.probability[index_counter]=prob
            index_counter+=1
```

Then, the probability of each outcome of the game is calculated by summing the probabilities of the relevant scores. 'sum_probs' is there just as a check (it should be close to 1).

```python
    p_win=0
    p_loss=0
    p_draw=0
    for i in range(len(probs)):
        if probs.homescore[i]>probs.awayscore[i]:
            p_win+=probs.probability[i]
        if probs.homescore[i]<probs.awayscore[i]:
            p_loss+=probs.probability[i]
        if probs.homescore[i]==probs.awayscore[i]:
            p_draw+=probs.probability[i] 
    season['p_win'][game]=p_win
    season['p_draw'][game]=p_draw
    season['p_loss'][game]=p_loss
    season['sum_probs'][game]=np.sum((p_win,p_draw,p_loss))
```

The expected value (EV) of a bet (full explanation [here](https://help.smarkets.com/hc/en-gb/articles/214554985-How-to-calculate-expected-value-in-betting)) is in the long run, how much profit you can expect to make per £1 placed bet. For example, if a bookie offered odds of 1.9 on a cointoss (with underlying probability of 50%) you’d start losing money fairly fast if you were to place a lot of bets ([if you're new to betting concepts](https://mybettingsites.co.uk/learn/betting-odds-explained/)). Therefore we want to calculate the EV, and then place bets in accordance.
```python
for game in range(len(season)):
    season.ev_win[game]=(season.p_win[game]*(season.B365H[game]-1))-(1-season.p_win[game])
    season.ev_draw[game]=(season.p_draw[game]*(season.B365D[game]-1))-(1-season.p_draw[game])
    season.ev_loss[game]=(season.p_loss[game]*(season.B365A[game]-1))-(1-season.p_loss[game])
```

Now onto the actual placing of bets. Below is a snapshot of the 'for' loop and 'if' statement. 'games' is normally set to the number of matches in the amended 'season' data, but can be shortened to test the code. 'ev_max' is to make sure we bet on the outcome with the highest expected value. A 'threshold' for the EV is defined to give a confidence level for placing a bet. Then if betting on a win for a given match is the highest EV, and it is higher than our threshold, a bet is placed. The next 'if/else' statement records the actual outcome and updates our bankroll appropriately. The 'print' statements are just there for checking that the loops behave as desired.

```python
for game in range(games):
    result=0
    ev_max=max(season.ev_win[game],season.ev_draw[game],season.ev_loss[game]) 
    if season.ev_win[game]==ev_max and season.ev_win[game]>threshold:
        team_bet=season.HomeTeam[game]
        if season.FTHG[game]>season.FTAG[game]:
            betvalue=wager*(season.B365H[game]-1)
            result='won'
            correct+=1
        else:
            betvalue=-wager
            result='lost'
            incorrect+=1
        bankroll+=betvalue
        print("Bet",season.HomeTeam[game],'vs',season.AwayTeam[game],':backed',team_bet)
        print('home scored',season.FTHG[game], 'away scored',season.FTAG[game])
        print('Bet',result)
        print('Bankroll=',bankroll)
```

Finally, the ending ROI and breakdown of bets are reported. 

# Discussion & Extensions
The model itself has very poor predictive power - likely becuase the goals are unlikely to be modelled by a Poisson this basic. The ROI is pretty terrible as the expected value of the bets is only as accurate as your model. 

I'm next hoping to implement the full Dixon-Coles model, and also build the code to back test over every season automatically rather than manually changing the 'season' input. The latter will probably be done by defining the loops I have as functions.
