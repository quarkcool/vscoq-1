#CoqTop XML Protocol#

This documentation aims to provide a "hands on" description of the XML protocol that coqtop and coqide use to communicate.
A somewhat out-of-date description of the async state machine is [documented here](https://github.com/ejgallego/jscoq/blob/master/notes/coq-notes.md). Typings for the protocol can be [found here](https://github.com/coq/coq/blob/trunk/ide/interface.mli#L222).


* [Commands](#commands)
  - [Add](#command-add)
  - [EditAt](#command-editAt)
  - [Init](#command-init)
  - [Goal](#command-goal)
  - [Status](#command-status)
  - [Query](#command-query)
  - [Evars](#command-evars)
  - [Hints](#command-hints)
  - [Search](#command-search)
  - [GetOptions](#command-getoptions)
  - [SetOptions](#command-setoptions)
  - [MkCases](#command-mkcases)
  - [StopWorker](#command-stopworker)
  - [PrintAst](#command-printast)
  - [Annotate](#command-annotate)
* [Feedback messages](#feedback)
  - [Added Axiom](#feedback-addedaxiom)
  - [Processing](#feedback-processing)
  - [Processed](#feedback-processed)
  - [Incomplete](#feedback-incomplete)
  - [Complete](#feedback-complete)
  - [GlobRef](#feedback-globref)
  - [Error](#feedback-error)
  - [InProgress](#feedback-inprogress)
  - [WorkerStatus](#feedback-workerstatus)
  - [File Dependencies](#feedback-filedependencies)
  - [File Loaded](#feedback-fileloaded)


Sentences: each command sent to CoqTop is a "sentence"; they are typically terminated by ".\s" (followed by whitespace or EOF).
Examples: "Lemma a: True.", "(* asdf *) Qed.", "auto; reflexivity."
In practice, the command sentences sent to CoqTop are terminated at the "." and start with any previous whitespace.
Each sentence is assigned a unique stateId after being sent to Coq (via Add).
States:
  * Processing: has been received by Coq and has no obvious syntax error (that would prevent future parsing)
  * Processed:
  * InProgress:
  * Incomplete: the validity of the sentence cannot be checked due to a prior error
  * Complete:
  * Error: the sentence has an error error

State ID 0 is reserved as 'null' or 'default' state. (The 'query' command suggests that it might also refer to the currently-focused state, but I have not tested this yet). The first command should be added to state ID 0. Queries are typically performed w.r.t. state ID 0.

--------------------------

## <a name="commands">Commands</a>

### <a name="command-add">**Add(stateId: integer, command: string)**</a>
```html
<call val="Add">
  <pair>
    <pair>
      <string>${command}</string>
      <int>${editId}</int>
    </pair>
    <pair>
      <state_id val="${stateId}"/>
      <bool val="false"/>
    </pair>
  </pair>
</call>
```

#### *Returns*
* The added command is given a fresh `stateId` and becomes the next "tip".
```html
<value val="good">
  <pair>
    <state_id val="${newStateId}"/>
    <pair>
      <union val="in_l"><unit/></union>
      <string>${message}</string>
    </pair>
  </pair>
</value>
```
* When closing a focused proof (in the middle of a bunch of interpreted commands),
the `Qed` will be assigned a prior `stateId` and `nextStateId` will be the id of an already-interpreted
state that should become the next tip. 
```html
<value val="good">
  <pair>
    <state_id val="${stateId}"/>
    <pair>
      <union val="in_r"><state_id val="${nextStateId}"/></union>
      <string>${message}</string>
    </pair>
  </pair>
</value>
```
* Failure:
  - Syntax error. Error offsets are with respect to the start of the sentence.
  ```html
  <value val="fail"
      loc_s="${startOffsetOfError}"
      loc_e="${endOffsetOfError}">
    <state_id val="${stateId}"/>
    ${errorMessage}
  </value>
  ```
  - Another error (e.g. Qed before goal complete)
  ```html
  <value val="fail"><state_id val="${stateId}"/>${errorMessage}</value>
  ```

-------------------------------

### <a name="command-editAt">**EditAt(stateId: integer)**</a>
#### *Returns*
* Simple backtrack; focused stateId becomes the parent state
```html
<value val="good">
  <union val="in_l"><unit/></union>
</value>
```

* New focus; focusedQedStateId is the closing Qed of the new focus; senteneces between the two should be cleared
```html
<value val="good">
  <union val="in_r">
    <pair>
      <state_id val="${focusedStateId}"/>
      <pair>
        <state_id val="${focusedQedStateId}"/>
        <state_id val="${oldFocusedStateId}"/>
      </pair>
    </pair>
  </union>
</value>
```
* Failure: If `stateId` is in an error-state and cannot be jumped to, `errorFreeStateId` is the parent state of ``stateId` that shopuld be edited instead. 
```html
<value val="fail" loc_s="${startOffsetOfError}" loc_e="${endOffsetOfError}">
  <state_id val="${errorFreeStateId}"/>
  ${errorMessage}
</value>
```

-------------------------------

### <a name="command-init">**Init()**</a>
* No options.
```html
<call val="Init"><option val="none"/></call>
```
* With options. Looking at [ide_slave.ml](https://github.com/coq/coq/blob/c5d0aa889fa80404f6c291000938e443d6200e5b/ide/ide_slave.ml#L355), it seems that `options` is just the name of a *.v file, whose path is added via `Add LoadPath` to the initial state.
```html
<call val="Init">
  <option val="some">
    <string>${options}</string>
  </option>
</call>
```

#### *Returns*
* The initial stateId (not associated with a sentence)
```html
<value val="good">
  <state_id val="${initialStateId}"/>
</value>
```

-------------------------------


### <a name="command-goal">**Goal()**</a>
```html
<call val="Goal"><unit/></call>
```
#### *Returns*
* If there is a goal. `backgroundGoals`, `shelvedGoals`, and `abandonedGoals` have the same structure as the first set of (current/foreground) goals. 
```html
<value val="good">
  <option val="some">
  <goals>
    <list>
      <goal>
        <string>3</string>
        <list>
          <string>${hyp1}</string>
          ...
          <string>${hypN}</string>
        </list>
        <string>${goal}</string>
      </goal>
      ...
      ${goalN}
    </list>
    ${backgroundGoals}
    ${shelvedGoals}
    ${abandonedGoals}
  </goals>
  </option>
</value>
```
* No goal:
```html
<value val="good"><option val="none"/></value>
```

-------------------------------


### <a name="command-status">**Status(force)**</a>
CoqIDE typically sets `force` to `false`. 
```html
<call val="Status"><bool val="${force}"/></call>
```
#### *Returns*
*  
```html
<status>
  <string>${path}</string>
  <string>${proofName}</string>
  <string>${allProofs}</string>
  <string>${proofNumber}</string>
</status>
```

-------------------------------


### <a name="command-query">**Query(query, stateId)**</a>
In practice, `stateId` is 0, but the effect is to perform the query on the currently-focused state.
```html
<call val="Query">
  <string>${query}</string>
  <state_id val="${stateId}"/>
</call>
```
#### *Returns*
*
```html
(TODO...)
```
-------------------------------



### <a name="command-evars">**Evars()**</a>
```html
<call val="Evars"><unit/></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-hints">**Hints()**</a>
```html
<call val="Hints"><unit/></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-search">**Search(...)**</a>
```html
<call val="Search">...</call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-getoptions">**GetOptions()**</a>
```html
<call val="GetOptions"><unit/></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-setoptions">**SetOptions(options)**</a>
Sends a list of option settings, where each setting roughly looks like:
`([optionNamePart1, ..., optionNamePartN], value)`.
```html
<call val="SetOptions">
  <list>
    <pair>
      <list>
        <string>optionNamePart1</string>
        ...
        <string>optionNamePartN</string>
      </list>
      <option_value val="${typeOfOption}">
        <option val="some">
          ${value}
        </option>
      </option_value>
    </pair>
    ...
    <!-- Example: -->
    <pair>
      <list>
        <string>Printing</string>
        <string>Width</string>
      </list>
      <option_value val="intvalue">
        <option val="some"><int>60</int></option>
      </option_value>
    </pair>
  </list>
</call>
```
CoqIDE sends the following settings (defaults in parentheses):
```
Printing Width : (<option_value val="intvalue"><int>60</int></option_value>),
Printing Coercions : (<option_value val="boolvalue"><bool val="false"/></option_value>),
Printing Matching : (...true...)
Printing Notations : (...true...)
Printing Existential Instances : (...false...)
Printing Implicit : (...false...)
Printing All : (...false...)
Printing Universes : (...false...)
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-mkcases">**MkCases(...)**</a>
```html
<call val="MkCases"><string>...</string></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-stopworker">**StopWorker(worker)**</a>
```html
<call val="StopWorker"><string>${worker}</string></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------


### <a name="command-printast">**PrintAst()**</a>
```html
<call val="PrintAst"><unit/></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------



### <a name="command-annotate">**Annotate(annotation)**</a>
```html
<call val="Annotate"><string>${annotation}</string></call>
```
#### *Returns*
*
```html
(TODO...)
```

-------------------------------

## <a name="feedback">Feedback messages</a>

Feedback messages are issued out-of-band,
  giving updates on the current state of sentences/stateIds,
  worker-thread status, etc.

* <a name="feedback-addedaxiom">Added Axiom</a>: in response to `Axiom`, `admit`, `Admitted`, etc.
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="addedaxiom" />
</feedback>
```
* <a name="feedback-processing">Processing</a>
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="processingin">
    <string>${workerName}</string>
  </feedback_content>
</feedback>
```
* <a name="feedback-processed">Processed</a>
```html
</feedback>
  <feedback object="state" route="0">
    <state_id val="${stateId}"/>
  <feedback_content val="processed"/>
</feedback>
```
* <a name="feedback-incomplete">Incomplete</a>
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="incomplete" />
</feedback>
```
* <a name="feedback-complete">Complete</a>
* <a name="feedback-globref">GlobRef</a>
* <a name="feedback-error">Error</a>. Issued, for example, when a processed tactic has failed or is unknown.
The error offsets may both be 0 if there is no particular syntax involved.
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="errormsg">
    <loc start="${sentenceOffsetBegin}" stop="${sentenceOffsetEnd}"/>
    <string>${errorMessage}</string>
  </feedback_content>
</feedback>
```
* <a name="feedback-inprogress">InProgress</a>
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="inprogress">
    <int>1</int>
  </feedback_content>
</feedback>
```
* <a name="feedback-workerstatus">WorkerStatus</a>
Ex: `workername = "proofworker:0"`
Ex: `status = "Idle"` or `status = "proof: myLemmaName"` or `status = "Dead"`
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="workerstatus">
    <pair>
      <string>${workerName}</string>
      <string>${status}</string>
    </pair>
  </feedback_content>
</feedback>
```
* <a name="feedback-filedependencies">File Dependencies</a>. Typically in response to a `Require`. Dependencies are *.vo files.
  - State `stateId` directly depends on `dependency`:
  ```html
  <feedback object="state" route="0">
    <state_id val="${stateId}"/>
    <feedback_content val="filedependency">
      <option val="none"/>
      <string>${dependency}</string>
    </feedback_content>
  </feedback>
  ```
  - State `stateId` depends on `dependency` via dependency `sourceDependency`
  ```html
  <feedback object="state" route="0">
    <state_id val="${stateId}"/>
    <feedback_content val="filedependency">
      <option val="some"><string>${sourceDependency}</string></option>
      <string>${dependency}</string>
    </feedback_content>
  </feedback>
  ```
* <a name="feedback-fileloaded">File Loaded</a>. For state `stateId`, module `module` is being loaded from `voFileName`
```html
<feedback object="state" route="0">
  <state_id val="${stateId}"/>
  <feedback_content val="fileloaded">
    <string>${module}</string>
    <string>${voFileName`}</string>
  </feedback_content>
</feedback>
```

