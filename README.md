# Artificial Intelligence 2026/2027 Course at FMI Sofia University

Repository with the exercises for the "Artificial Intelligence" course given by me
@ Faculty of Mathematics and Informatics, Sofia University.

## Evaluation

- Ongoing Assessment (50%)
  - Test 1 ≈ ⅓ (at least 3.00)
  - Test 2 ≈ ⅓ (at least 3.00)
  - Homeworks ≈ ⅓ (at least 3.00 from at least 5 tasks)

- On-Site Real-Time Programming Assessment
  - Twice, once per part, with at least 1 Pass required.

- Exam (50%)
  - Project Presentation ≈ ⅔ (optional)
  - Interview (final assessment) ≈ ⅓
  - The evaluation is more like a point system - for excellent you do not need to collect the maximum number of points.

## Homeworks

### How many you need

The grade depends on how many homeworks are **perfect**, not on how many you hand in.

| Perfect homeworks | Grade         |
| ----------------: | ------------- |
|            0 to 4 | Poor 2        |
|                 5 | Very Good 5   |
|                 6 | Excellent 6   |
|                 7 | Excellent 6+  |
|         8 or more | Excellent 6++ |

Read the first row carefully: four perfect homeworks and a fifth that is almost right still comes
out as Poor 2 on this table. Five is the number that matters.

Homeworks that are good but not perfect are graded differently - five or six of them usually average
to Sufficient 3, Good 4 or Very Good 5, and occasionally to Poor 2 or Excellent 6. So the table is
the ceiling, not a formula.

### Rules

- Write them in any language you like: C, C++, Java, C#, Python, or anything else.
- **Implement everything from scratch.** Libraries providing data structures are fine; libraries
  implementing the algorithm, the data splitting, or the evaluation are not.

### Deadlines and defences

- Part I covers topics 1 to 5, and is due within 2 weeks after topic 5.
- Part II covers topics 6 to 10, and is due within 2 weeks after topic 10.
- You defend up to 3 homeworks per part - 2+3, 3+2 or 3+3.
- Defences happen only in the designated sessions, one after each part, and all of them must be
  finished **within the semester**. None are held during the exam session.
- Each part also includes an on-site real-time programming assessment.
- Everything must be defended by **24.01.2027**. A homework not presented within the semester may
  not be accepted afterwards, and that can mean a Poor 2.

### Automatic testing

Check your solution against the official tests before submitting it, with the console tool
[fmi-ai-judge](https://pypi.org/project/fmi-ai-judge/).

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install fmi-ai-judge
```

It does not compile anything - it runs an executable or a script you already built.

```bash
judge list                                  # the problems it knows
judge run -p frog-leap ./main.out           # run one solution
judge run -p frog-leap --exec "python3 {src}" solution.py
```

Covered so far: `frog-leap`, `n-puzzle`, `n-queens`, `tsp`, `knapsack`, `tic-tac-toe`. Each accepts
short aliases, and `-p` can be omitted when the file name already contains one.

Two things that catch people out. The input arrives on **standard input**, not as a command-line
argument. And the checker compares your output line by line, so anything extra - a timing line, a
prompt, a trailing summary - fails the test. Lines beginning with `#` are ignored, which is how the
optional `# TIMES_MS: alg=<milliseconds>` header works together with `--bench`.

The default limit is one second per test, but the judge is more forgiving than that sounds. It
detects the runtime and gives scripts the slow tier automatically, it measures interpreter start-up
separately (the `cal` column) so that it is not charged against you, and a stress test can carry its
own larger limit. `--slow` forces the slow tier for everything. Results are written to
`.judge/results.json` and `.judge/results.csv`.

## Exercises' Topics

10 𝑇𝑜𝑝𝑖𝑐𝑠 = 𝑓𝑟𝑜𝑚 Homeworks 8 𝑡𝑜 10

- Problem Solving and Search
  - Uninformed _(Blind)_ Search
  - Informed _(Heuristic)_ Search
  - Constraint Satisfaction Problems
  - Genetic Algorithms
  - Games
- Machine Learning
  - _k_ - Nearest Neighbors
  - Naïve Bayes Classifier
  - Decision Tree
  - _k_ Means
  - Neural Networks

## Exercises by Week

1. Introduction, Recap and Uninformed (Blind) Search
2. Informed (Heuristic) Search
3. Constraint Satisfaction Problems
4. Genetic Algorithms
5. Games
6. Introduction to Machine Learning
7. k-Nearest Neighbors
8. Naïve Bayes Classifier
9. Decision Tree
10. kMeans
11. Neural Networks

## Content by weeks

### 0. [Python Intro](./00_python_intro)

### 1. [Uninformed (Blind) Search](./01_blind_search)

### 2. [Informed (Heuristic) Search](./02_informed_search)

### 3. [Constraint Satisfaction Problems](./03_constraint_satisfaction)

### 4. [Genetic Algorithms](./04_genetic_algorithms)

### 5. [Adversarial Search and Games](./05_games)

### 6. [Linear and Logistic Regression](./06_linear_and_logistic_regression)

### 7. [k-Nearest Neighbours and Model Evaluation](./07_kNN_model_evaluation)

### 8. [Naive Bayes](./08_naive_bayes)

### 9. [Decision Trees and Ensembles](./09_decision_trees)

## Environment

The course Python version is **3.12**. Every library needed to run the notebooks is listed in
[requirements.txt](./requirements.txt).

Pick whichever of the two setups below you prefer. Both end with the same thing: an isolated
environment with the course packages installed, and a Jupyter kernel pointing at it.

### Option 1: pyenv + venv (macOS / Linux)

`pyenv` manages Python versions, and the built-in `venv` module manages the packages. No extra
plugins needed.

1. Install pyenv.

   macOS:

   ```
     brew install pyenv
   ```

   Linux:

   ```
     curl -fsSL https://pyenv.run | bash
   ```

   On Debian/Ubuntu you also need the headers Python is built against, otherwise the build
   silently drops modules such as `sqlite3` and `lzma`:

   ```
     sudo apt update && sudo apt install -y build-essential libssl-dev zlib1g-dev \
       libbz2-dev libreadline-dev libsqlite3-dev libffi-dev liblzma-dev tk-dev
   ```

2. Add pyenv to your shell, then restart it.

   zsh (the default on macOS):

   ```
     echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
     echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
     echo 'eval "$(pyenv init - zsh)"' >> ~/.zshrc
     exec "$SHELL"
   ```

   bash:

   ```
     echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
     echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
     echo 'eval "$(pyenv init - bash)"' >> ~/.bashrc
     exec "$SHELL"
   ```

3. Install Python 3.12 and pin it for this repository. Giving `pyenv install` just `3.12` picks
   the newest 3.12.x; `pyenv install --list` shows everything available.

   ```
     pyenv install 3.12
     cd ArtificialIntelligence2026
     pyenv local 3.12
   ```

   `pyenv local` writes a `.python-version` file, so every command you run inside this directory
   uses 3.12 from now on.

4. Create and activate the virtual environment:

   ```
     python -m venv .venv
     source .venv/bin/activate
   ```

   Your prompt should now start with `(.venv)`. Leave the environment later with `deactivate`.

5. Install the libraries:
   ```
     pip install --upgrade pip
     pip install -r requirements.txt
   ```

Both `.venv/` and `.python-version` are ignored by git, so your setup stays local to your machine.

### Option 2: conda (any platform)

[Conda](https://docs.conda.io/projects/conda/en/latest/index.html#) manages the Python version and
the packages together.

1. To create an environment:

   ```
     conda create --name <my-env> python=3.12
   ```

   Replace `<my-env>` with the name of your environment.

2. When conda asks you to proceed, type `y`:

   `proceed ([y]/n)?`

   This creates the environment in `/envs/`. No packages will be installed in it yet.

3. Then you need to activate your environment:

   ```
     conda activate <my-env>
   ```

4. Then you can install the necessary libraries:
   ```
     pip install -r requirements.txt
   ```

### Register the Jupyter kernel

With your environment activated, make it selectable from inside the notebooks:

```
  python -m ipykernel install --user --name ai-2026 --display-name "Python (AI 2026)"
```

Then open a notebook and choose **Python (AI 2026)** as the kernel. If imports fail even though
`pip install` succeeded, this is almost always the reason -- the notebook is running against a
different interpreter than the one you installed into. `import sys; print(sys.executable)` in a
cell tells you which one it is using.

## Resources

- The Main Book

- Stuart Russell and Peter Norvig. Artificial Intelligence: A Modern Approach. Prentice Hall. [http://aima.eecs.berkeley.edu/](http://aima.eecs.berkeley.edu/)

- Lectures, Exercises and all other resources at Moodle
