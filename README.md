# 🎡 Lucky Roulette

Done with the help of WiseMystical-Tree, this is a desktop prize roulette written in Python, with a built-in SQLite database. After every spin, the program uses the history of previous plays to **recalculate the probability of each prize**, so prizes that come out too often become less likely and rare ones become more likely.

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Configuring the Prizes](#configuring-the-prizes)
- [Database](#database)
- [Customization](#customization)
- [Project Structure](#project-structure)
- [License](#license)

## Features

- Animated roulette wheel with a casino-style look, built with Tkinter (runs fullscreen; press `Esc` to leave fullscreen).
- Weighted random draw: each prize has its own weight, which decides how likely it is to come out.
- **Adaptive probabilities**: weights are recalculated after every play, based on how often each prize has actually been won.
- Each prize has a **minimum and a maximum weight**, so probabilities never drift too far from the original design.
- Prizes are managed in a simple CSV file, with no code changes needed to add, edit or remove them.
- Full play history stored in SQLite, including every weight change.
- Optional email identification, with a limited number of attempts per participant.
- Special prize **"Tente novamente"** (try again), which gives the participant an extra attempt.
- No external dependencies.

## How It Works

### Weights, not percentages

Weights are **relative**: they do not need to add up to 100. The higher the weight, the more likely the prize is to come out. The chance of a prize is its current weight divided by the sum of all current weights.

With the default prizes, the initial chances are:

| Prize            | Base weight | Initial chance |
| ---------------- | ----------: | -------------: |
| Caneta           |          20 |           8.7% |
| Lápis            |          10 |           4.3% |
| Moldura          |          25 |          10.9% |
| Fita             |           5 |           2.2% |
| Tente Novamente  |          65 |          28.3% |
| Sem Prémio       |          80 |          34.8% |
| Cupão Worten     |          15 |           6.5% |
| Cupão Fnac       |          10 |           4.3% |

### Recalculation after each play

After every spin, the program recalculates the weight of each active prize:

1. It computes how many times the prize **should** have come out so far, given its base weight:
   `expected = total_plays × (base_weight / sum_of_base_weights)`
2. It compares this with how many times it **actually** came out (`real`).
3. It measures the relative deviation: `deviation = (real - expected) / max(expected, 1)`
4. It computes an adjustment factor, limited to the range **0.70 – 1.30**:
   `factor = 1 - 0.25 × deviation`
5. The new weight is `base_weight × factor`, always clamped between the prize's `peso_min` and `peso_max`.

In short: a prize that came out **more** than expected loses weight, and a prize that came out **less** than expected gains weight. If there are no plays yet, all weights go back to their base value.

Every change is recorded in the `historico_pesos` table together with the reason.

## Requirements

- Python 3.14 (developed and tested with 3.14.4)
- Tkinter (included with the standard Python installers for Windows and macOS; on some Linux distributions it must be installed separately, e.g. `sudo apt install python3-tk`)

No `pip install` is needed: the project only uses the standard library (`sqlite3`, `tkinter`, `csv`, `random`, `re`, `math`, `pathlib`).

## Getting Started

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/<your-repository>.git
cd <your-repository>

# 2. Create your prize file (see the next section)

# 3. Run the program
python Roleta_da_Sorte.py
```

The SQLite database file is created automatically the first time the program runs.

## Configuring the Prizes

Prizes are defined in a file called `premios.csv`, placed in **the same folder as the script**. If the file does not exist, the roulette has no prizes and the program shows an error.

Edit the file by hand with any text editor. Each line describes one prize:

```csv
nome,peso_base,peso_min,peso_max
Caneta,20,15,23
Lápis,10,8,12
Moldura,25,20,30
Fita,5,3,7
Tente Novamente,65,60,70
Sem Prémio,80,75,90
Cupão Worten,15,10,20
Cupão Fnac,10,5,15
```

| Column      | Required | Description                                                        |
| ----------- | :------: | ------------------------------------------------------------------ |
| `nome`      |    ✅    | Prize name (must be unique, case-insensitive).                     |
| `peso_base` |    ✅    | Starting weight and reference value for the recalculation.         |
| `peso_min`  |    ✅    | Lowest weight the prize can ever have.                             |
| `peso_max`  |    ✅    | Highest weight the prize can ever have.                            |
| `cor`       |    ❌    | Segment color in hex (e.g. `#FF8800`). A random color if omitted.  |

Rules and tips:

- Weights must satisfy `peso_min <= peso_base <= peso_max`.
- Decimal numbers can use either a dot or a comma (`2.5` or `2,5`).
- Lines starting with `#` are ignored (comments).
- An optional first line containing only `1` or `0` turns the email request on (`1`, the default) or off (`0`). If used, the column header goes on the second line.
- The prize called `Tente Novamente` is special: when it comes out, the participant gets one more attempt.

### Applying changes

The file is the **source of truth** for the prizes. It is read when the program starts, before every spin, and whenever you click **"Recarregar prémios"** in the interface:

- A prize in the file but not in the database is **added**.
- A prize in both is **updated** (the current weight is kept, but adjusted if it falls outside the new limits).
- A prize in the database but no longer in the file is **deactivated**, not deleted, so past plays are preserved.

## Database

The program uses a local SQLite database (`Projeto_Roleta_teste.db` by default) with four tables:

| Table             | Purpose                                                                       |
| ----------------- | ----------------------------------------------------------------------------- |
| `premios`         | Prizes, with base, current, minimum and maximum weight, color and active flag. |
| `participantes`   | Participants (by email), remaining attempts and total number of plays.        |
| `jogadas`         | Every play: participant, prize won, weight used and date/time.                |
| `historico_pesos` | History of every weight change and the reason for it.                         |

You can inspect the data with any SQLite tool, for example [DB Browser for SQLite](https://sqlitebrowser.org/).

## Customization

The main settings are constants at the top of the script:

| Constant                  | Default                         | Description                                          |
| ------------------------- | ------------------------------- | ---------------------------------------------------- |
| `NOME_BD`                 | `Projeto_Roleta_teste.db`       | Name of the SQLite database file.                    |
| `FICHEIRO_PREMIOS_CSV`    | `premios.csv`                   | Path of the prize file.                              |
| `SOLICITAR_EMAIL`         | `True`                          | Ask for an email before playing (the prize file's first line overrides this). |
| `EMAIL_PADRAO_SEM_LOGIN`  | `anonimo@roleta.local`          | Email used when the email request is turned off.     |

## Project Structure

```text
.
├── Roleta_da_Sorte.py   # Application (GUI, database and probability logic)
├── premios.csv          # Prize definitions (edit by hand)
├── README.md
└── Projeto_Roleta_teste.db   # Created automatically on first run
```

## License

Copyright © 2026. All rights reserved.

No license is granted for this project. Please contact the author before copying, modifying or redistributing any part of it.
