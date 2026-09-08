# AiTournament
Code to run AI tournaments for scarce resource division, such as the [MAD Chairs game](https://arxiv.org/abs/2503.20986) (a.k.a. the [Lifeboat Problem](https://www.sciencedirect.com/science/article/pii/S0014292111001231?casa_token=Lk71pF1iix4AAAAA:4tvPXdyzSc6Nh0UzddxhizMqx3xNA1U7PSeArEcQvWUUD32aH2duyonpB1DqeHfiBXhw-7Vy4i4)). Such games are a metaphor for real-world division of scarce resources including division of jobs, hospital beds, and spaces in traffic. One important application is the division of opportunity to have voice in a conversation. Representative democracies, for example, explicitly establish a limited set of channels through which citizens can have voice in their government, and any of those channels can be overcrowded.

[Other software](https://github.com/ChrisSantosLang/MADChairs) is available to run such games with human players (or mixes of human and AI players). This software is much faster for situations in which all players are AI. 

## Installation
You can open this code in [Google Colab](https://colab.research.google.com/) by following [this link](https://colab.research.google.com/github//ChrisSantosLang/AiTournament/blob/main/MADChairs.ipynb). Click "Run all". The output files will be generated in the Files tab (it takes about 3 minutes). 

The first cell, containing `!pip install trueskill`, must be run once to initialize the environment, but subsequent runs can skip that step (i.e. use the run button for the second cell). 

## Expected outputs
MAD Chairs is a game repeated for multiple rounds. In each round, each player selects from a set of resources (e.g. "A", "B", "C", "D" or "E") or selects to "skip". Each player who selects a resource no other player selects for that round wins that round.

Running the code will output three files:
 * `{strategy_name}_results.csv` shows what the players selected in each round of each match and how often they won (as a %).
 * `{strategy_name}_stats.csv` shows how well each strategy performed against the other strategies.
 * `{strategy_name}_submission.csv` is the file to submit for a Kaggle contest. It contains only the "skill" rating for your submission (see calculation below).

The stats file shows the "edge" and win rate of several strategies against each other. The standard competing strategies include

 * **random** selects randomly.
 * **rotate0** selects a unique resources for each of the first players and skip to the rest and continues those assignments indefinitely.
 * **random3** selects like **random** in round 1. Afterwards, it always repeats its previous selection when it won but only 1/3 of the time when it lost. If changing, it randomly selects from the resources selected least in the previous round.
 * **rotate** selects like **rotate0** in round 1. Afterward, assignments rotate by one position in each round.
 * **radicaleq** assigns the top resource to the player who has won the least (i.e. the least wealthy), then the next resource for the next wealthiest,  etc. After each resource has been assigned, all remaining players are assigned to 'skip'. In the case of ties, the player with later position counts as wealthier.
 * **equalize** is like **radicaleq**, but ingroup/outgroup accounting is added: It maintains a count of deviations from the strategy discounted by 30% per round (so deviations in the distant past will be forgiven). Any   player who has deviated at least once (after discounting) is in the outgroup and counted as wealthier than everyone in the ingroup.
 * **caste** uses the [Trueskill](https://github.com/sublee/trueskill) algorithm to maintain skill-estimates for all players, uses those estimates to predict probabilities of winning, and maintains accounts of favors owed between all players, where debt incurred from beating a player is the probability of that other player winning and debt incurred from tying is that same probability minus one's own probability of winning. Returns as with equalize(), substituting credit for wealth.
 * **turntaking** selects like **caste**, but substituting debt for credit.

"Edge" is a measure of incentive to defect. The edge of A against B is the lowest win rate among players following A minus the lowest win rate among players following B. Half of the edge numbers are left out below because they effectively duplicate the displayed numbers (i.e. edge_A_vs_B = -edge_B_vs_A).

![Edge and win rates in 3v3 MAD Chairs](https://github.com/ChrisSantosLang/AiTournament/blob/main/Media/3v3madchairs.png?raw=true)

To the extent that players are rational, they are likely to defect from B to A if A has clear edge over B, so the win rate of A against B is expected to converge toward the win rate of A against itself. Likewise, the win rate of A against B is expected to converge toward zero if B has clear edge over A (i.e. A has negative edge against B). We call this use of the edge statistic to modify win rate "ewin", where ewin_A_vs_B = min(mean_win_A_vs_A, mean_win_A_vs_B * (rationality ** edge_A_vs_B)). 

When rationality is 1, edge has no impact on ewin. As rationality rises, edge dominates ewin (except when edge is 0). We use 50 for this software by default. The winner of each match is the strategy with the higher ewin. These wins are combined across many matches to generate a skill estimate for each strategy using the [Trueskill](https://github.com/sublee/trueskill) algorithm which has been ranking players on XBox since 2005 (like Elo rating in chess). The final reported score is the normalized trueskill score relative to the "turntaking" strategy.

## Modifying the code
The main way to modify this code is to fill the `submission()` function with your own MAD Chairs strategy. The parameters include `position: int`, `round: int`, `history: pd.DataFrame`, and `cache: dict`. The function  should return 'skip' or the letter of the resource your strategy would recommend to a player in the given `position` (1-6) and `round` (1-20) with the given `history` (which contains columns for "Position" and "Round{n}").

The cache may be used to improve efficiency by storing values calculated in previous calls to submission(). For examples, see `randomN()`, `radicaleq()`, `equalize()`, `caste()` and `turntaking()` in the same notebook.

Under `#Constants` you may also find it productive to modify:

 * `schedule` The code on this repository is set to pull `schedule.csv` from this github repository, but you can replace the url with the path to a local file instead. When debugging, a smaller file can help speed feedback. 
 * `rounds` (default `20`) to explore the impacts of greater/lesser iteration
 * `resources` (default `["A", "B", "C", "D", "E"]`) perhaps to make resources even more scarce
 * `num_players` (default `6`) if using a schedule file specifying matches for a different number of players
 * `rationality` (default `50`) to explore different assumptions about rationality

## Sample files
In addition to the `schedule.csv` file which specifies the matches of a tournament, this github repository includes:

 * `alwaysA_results.csv`: A sample results output. It was generated using `return "A"` for `submission()`.
 * `alwaysA_stats.csv`: A sample stats output. It was generated using `return "A"` for `submission()`.
 * `alwaysSkip_submission.csv`: A sample submission output. It was generated using the strategy of always skipping.
