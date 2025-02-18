
--8<-- "includes/common.md"

# Verdict Explainability
---

## An example

Let us briefly review the property `prop_no_leak.hml`, that expresses:

> (P~3~)&nbsp;&nbsp;&nbsp;&nbsp;The token `#!erlang 1` is *not* leaked by the server in reply to a client request.

```maxhml linenums="1"
with
  token_server:loop(_, _)
check
  [{_ <- _, token_server:loop(OwnTok, _)}]
  max(X.
    [{_ ? _}](
      [{_:_ ! Tok when OwnTok =:= Tok}]ff
      and
      [{_:_ ! Tok when OwnTok =/= Tok}]X
    )
  ).
```

The corresponding high-level description of the monitor generated by `#!erlang maxhml_eval:compile/2` is as follows.

```maxhml linenums="1"
{_ <- _, token_server:loop(OwnTok, _)}.rec X(
  {_ ? _}(
      {_:_ ! Tok when OwnTok =:= Tok}.no
      +
      NOT*{_:_ ! Tok when OwnTok =:= Tok}*.yes
    and
      {_:_ ! Tok when OwnTok =/= Tok}.X
      +
      NOT*{_:_ ! Tok when OwnTok =/= Tok}*.yes
  )
  +
  NOT*{_ ? _}*.yes
)
+
NOT*{A <- B, token_server:loop(OwnTok, _)}*.yes
```

In this high-level description, we use `+` to denote a mutually-exclusive choice between two branches of the monitor, and `NOT* ... *` to designate the monitor branch where *(i)* either the specified action pattern does not match the actual event from the trace, or, *(ii)* the pattern matches, but the Boolean constraint is not satisfied, given the data inside the variables contained in the pattern.
Parallel conjunctions are written as `and`, recursion is denoted by `rec`, and `yes` and `no` respectively denote the verdicts that monitors flag.


# Monitor reduction, step-by-step

Recall the monitor:

```maxhml linenums="1"
{_ <- _, token_server:loop(OwnTok, _)}.rec X(
  {_ ? _}(
      {_:_ ! Tok when OwnTok =:= Tok}.no
      +
      NOT*{_:_ ! Tok when OwnTok =:= Tok}*.yes
    and
      {_:_ ! Tok when OwnTok =/= Tok}.X
      +
      NOT*{_:_ ! Tok when OwnTok =/= Tok}*.yes
  )
  +
  NOT*{_ ? _}*.yes
)
+
NOT*{A <- B, token_server:loop(OwnTok, _)}*.yes
```

Let us assume that our *buggy* version of the token server exhibited the execution below. 
For this sequence of events, we describe the monitor reduction that explains how a verdict is reached.

```erlang linenums="1"
{trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,1]}}
{trace,<0.134.0>,'receive',{<0.136.0>,0}}
{trace,<0.134.0>,send,1,<0.136.0>}
```

1.  The monitor analyses the event `#!erlang {trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,2]}}` and it chooses the left  branch `{A <- B, token_server:loop(OwnTok, _)}` since the pattern matches, and its Boolean constraint `#!maxhml true` (elided) is always satisfied.
    The residual monitor with the instantiated data of `#!erlang OwnTok = 1`.

    ```maxhml linenums="1"
    rec X(
      {_ ? _}(
          {_:_ ! Tok when 1 =:= Tok}.no
          +
          NOT*{_:_ ! Tok when 1 =:= Tok}*.yes
        and
          {_:_ ! Tok when 1 =/= Tok}.X
          +
          NOT*{_:_ ! Tok when 1 =/= Tok}*.yes
      )
      +
      NOT*{_ ? _}*.yes
    )
    ```

2.  It then performs an internal transition to unfold the recursive variable `#!maxhml X` (expansion is not shown in the monitor due to space reasons).

    ```maxhml linenums="1"
    {_ ? _}(
        {_:_ ! Tok when 1 =:= Tok}.no
        +
        NOT*{_:_ ! Tok when 1 =:= Tok}*.yes
      and
        {_:_ ! Tok when 1 =/= Tok}.X
        +
        NOT*{_:_ ! Tok when 1 =/= Tok}*.yes
    )
    +
    NOT*{_ ? _}*.yes
    ```

3.  It analyses `#!erlang {trace,<0.134.0>,'receive',{<0.136.0>,0}}`, and choosing the left monitor branch.
    The left monitor branch matches any receive event, which means that the right branch matches no receive event.

    ```maxhml linenums="1"
      {_:_ ! Tok when 1 =:= Tok}.no
      +
      NOT*{_:_ ! Tok when 1 =:= Tok}*.yes
    and
      {_:_ ! Tok when 1 =/= Tok}.X
      +
      NOT*{_:_ ! Tok when 1 =/= Tok}*.yes
    ```

4.  At this point, the two conjuncted sub-monitors can evolve in parallel, given the trace event `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.
    The left sub-monitor `#!maxhml {_:_ ! Tok when 1 =:= 1}.no + NOT*{_:_ ! Tok when 1 =:= 1}*.yes` chooses the left branch since the pattern matches and the Boolean constrain is fulfilled.
    The right sub-monitor `{_:_ ! Tok when 1 =/= 1}.X + NOT*{_:_ ! Tok when 1 =/= 1}*.yes` chooses the right branch since the pattern matches, but the Boolean constrain for the right branch is satisfied.
    This reduces the monitor to:

    ```maxhml linenums="1"
      no
    and
      yes
    ```

    The last internal computation leads the monitor to the rejection verdict `#!maxhml no`.

## Try this for yourself

detectEr Linear Time includes the function `#!erlang lin_analyzer:analyze_trace/2` to reduce a monitor, given a trace of events.
In this example, we use the monitor that we synthesised earlier for P~3~.
This is located in the script `props/prop_no_leak.hml`.

1.  First we invoke the main monitor function `#!erlang prop_no_leak:mfa_spec/1` to obtain the monitor instance.
    Since we need to match the pattern expected by `#!erlang mfa_spec`, *i.e.*, `#!erlang {token_server,loop,[_,_]}`, we can use any arguments we wish.
    In this example, we shall use `#!erlang 1` for both tokens.

    ```erl
    1> {ok, Mon} = prop_no_leak:mfa_spec({token_server,loop,[1,1]}).
    ...
    ```

    Note that `#!erlang prop_no_leak:mfa_spec/1` returns the tuple tagged with `#!erlang ok`, and we unwrap it accordingly to obtain the monitor in `#1erlang Mon`.

2.  Next, we create the trace to feed to the monitor as a list of events:

    ```erl
    2> Trc = [{trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,1]}},
    {trace,<0.134.0>,'receive',{<0.136.0>,0}},
    {trace,<0.134.0>,send,1,<0.136.0>}].
    ```

3.  Finally the trace can be analyzed using the following invocation:

    ```erl
    3> {PdList, _}= lin_analyzer:analyze_trace(Trc, Mon).
    ```
    Variable `#!erlang PdList` contains the whole justification that details the steps taken by the monitor to reach the violation verdict `no`.

4.  The `#!erlang lin_analyzer` module also provides the means to pretty print the contents of the proof derivation list `#!erlang PdList`.

    ```erl
    4> lin_analyzer:show_pdlist(PdList).
    ```

Step `4` shows that the monitor is reduced using the following operational semantic rules:

1.  Rule *mChsL* on `#!erlang {trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,1]}}`.
  
    1.1. Axiom *mAct* on `#!erlang {trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,1]}}`.

2.  Axiom *mRec* on `tau`.

3.  Rule *mChsL* on `#!erlang {trace,<0.134.0>,'receive',{<0.136.0>,0}}`.

    3.1. Axiom *mAct* on `#!erlang {trace,<0.134.0>,'receive',{<0.136.0>,0}}`.

4.  Rule *mPar* on `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.

    4.1. Rule *mChsL* on `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.

    4.1.1. Axiom *mAct* on `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.

    4.2. Rule *mChsR* on `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.

    4.2.1. Axiom *mAct* on `#!erlang {trace,<0.134.0>,send,1,<0.136.0>}`.

5.  Rule *mConYR* on `tau`.

For the complete reference of these operational rules, please refer to the companion paper.





<!-- 

# Demo video.



## Part 1. LOG LEVEL = TRACE

1. Download detectEr and compile it.

2. Open the examples folder and compile them.

3. Try a simple test without the instrumented monitor, send a request, not commenting on whether it works or not.

4. State the property in english, and write it in the text file. 

5. Write the property in maxHml in the same text file.

6. Compile the property, and comment that for the sake of brevity, we shall not go over the file.

7. Instead, open the website and show the monitor in high-level format.

8. Go back to the console and instrument the monitor.

9. Run the server again and this time it should give a bug.

10. Comment that we run the system with full logs, which enables us to see what happens, but it may be a bit difficult to parse the text.

11. What we can do instead is the save a copy of the trace and examine it offline.



## Part 2.

1. Create the trace on the console, saying that it was previously saved.

2. Load the monitor.

3. Analyse the trace.

4. Print proof derivation list.

5. Explain the proof derivation list and point to the last line where we have the variables x, y, and z populated and draw the attention to the 1.

6. Fix the bug and recompile.

7. Restart the shell and then re-instrument the monitor.

8. Say that we need not regenerate the monitor, since the instrumentation aspect and the monitor are in separate modules, and this gives us the advantage to re-instrument when needed without altering the monitor, or keep the instrumentation in place and regenerate the monitor, so long as the entry function does not change.

9. Run the token server, make a request and no violation should happen.

10. Comment on the unfolding of the recursive variable.



## Part 3 (Switch to the coordination_2020 directory).  LOG LEVEL = DEBUG

1. Say that we generated the formula in the paper that detects when a request is ok or no in Cowboy for our REST web service.

2. Show the formula written down.

3. Say that we compiled the formula down to a monitor, and we weaved Cowboy. Weave it, but remove the frames.

4. Start Cowboy forever.

5. Switch to the postman and make a bad request and show the violation. Make a good request if there is time.

6. In practice, it might be a bit difficult to go through to the logs, but this depends on the application. For instance here we have a lot of request data passed in each request.

7. We would like to explore this further, and try to see whether it is possible to present the data in a more human consumable format.

8. Grab the sanitised trace, and pass it through the thing, if there is time. But probably not.





-----------

{ok, Mon} = prop_no_leak:mfa_spec({token_server,loop,[1,1]}).

Trc = [{trace,<0.84.0>,spawned,<0.82.0>,{token_server,loop,[1,1]}},{trace,<0.134.0>,'receive',{<0.136.0>,0}},{trace,<0.134.0>,send,1,<0.136.0>}].



{PdList, _}= lin_analyzer:analyze_trace(Trc, Mon).


lin_analyzer:show_pdlist(PdList). -->