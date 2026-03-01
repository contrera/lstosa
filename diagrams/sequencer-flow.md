# Sequencer flow (src/osa/scripts/sequencer.py)

mermaid
flowchart TD
    A[main()] --> B[sequencer_cli_parsing()]
    B --> C{options.verbose?}
    C -->|yes| C1[log.setLevel(DEBUG)]
    C -->|no| C2[log.setLevel(INFO)]
    C1 --> D[tag = gettag()]
    C2 --> D
    D --> E[start(tag)]
    E --> F{options.tel_id in ["LST1","LST2"]?}
    F -->|yes| G[single_process(options.tel_id)]
    F -->|no| H[log.error("Process mode not supported yet")]

    G --> I[database = cfg.get("database","path")]
    I --> J{database?}
    J -->|yes| J1[osadb.start_processing(date_to_iso(options.date))]
    J -->|no| K
    J1 --> K[init: sequence_list, options.tel_id, options.directory, options.log_directory]
    K --> L{not options.simulate?}
    L -->|yes| L1[os.makedirs(options.log_directory, exist_ok=True)]
    L -->|no| M
    L1 --> M[summary_table = run_summary_table(options.date)]
    M --> N{len(summary_table)==0?}
    N -->|yes| N1[log.warning + sys.exit(0)]
    N -->|no| O{(not options.no_gainsel) and (not GainSel_finished(options.date))?}
    O -->|yes| O1[log.info + sys.exit()]
    O -->|no| P{is_day_closed()?}
    P -->|yes| P1[log.info + return sequence_list]
    P -->|no| Q{not options.test and not options.simulate?}

    Q -->|no| R
    Q -->|yes| Q1{is_sequencer_running(options.date)?}
    Q1 -->|yes| Q1a[log.info + sys.exit(0)]
    Q1 -->|no| Q2{is_sequencer_completed(options.date) and not options.force_submit?}
    Q2 -->|yes| Q2a[log.info + sys.exit(0)]
    Q2 -->|no| Q3{timeout_in_sequencer(options.date) and not options.force_submit?}
    Q3 -->|yes| Q3a[log.info + sys.exit(0)]
    Q3 -->|no| R

    R[sequence_list = build_sequences(options.date)] --> S[prepare_jobs(sequence_list)]
    S --> T[update_job_info(sequence_list)]
    T --> U[get_veto_list(sequence_list)]
    U --> V[get_closed_list(sequence_list)]
    V --> W[update_sequence_status(sequence_list)]
    W --> X{not options.no_submit?}
    X -->|yes| X1[submit_jobs(sequence_list)]
    X -->|no| Y
    X1 --> Y[report_sequences(sequence_list)]
    Y --> Z[return sequence_list]

    %% update_job_info
    T --> TJ{options.test?}
    TJ -->|yes| TJ1[return]
    TJ -->|no| TJ2[sacct_output, squeue_output = run_sacct(), run_squeue()]
    TJ2 --> TJ3[set_queue_values(...)]

    %% update_sequence_status
    W --> US[for each seq in seq_list]
    US --> U1{seq.type == "PEDCALIB"?}
    U1 -->|yes| U1a[seq.calibstatus = int(status(CALIB)*100/subruns)]
    U1 -->|no| U2{seq.type == "DATA"?}
    U2 -->|yes| U2a[set dl1status, dl1abstatus, datacheckstatus, muonstatus, dl2status]
    U2a --> U2b[seq.catbstatus = check_catB_status(seq)]

    %% check_catB_status
    U2b --> CS{check_catB_status(seq)}
    CS --> CS1[closed_files = directory.glob("catB*{run}*.closed")]
    CS1 --> CS2{closed_files?}
    CS2 -->|yes| CS2a[return "CLOSED"]
    CS2 -->|no| CS3[log_files = log_directory.glob("catB_calibration_{run}_*.err")]
    CS3 --> CS4{log_files?}
    CS4 -->|no| CS4a[return "None"]
    CS4 -->|yes| CS5[parse job_id from filename]
    CS5 --> CS6[sacct_info = get_sacct_output(run_sacct(job_id))]
    CS6 --> CS7{sacct_info empty?}
    CS7 -->|yes| CS4a
    CS7 -->|no| CS8[return sacct_info.iloc[0]["State"]]

    %% get_status_for_sequence
    U2a --> GS{get_status_for_sequence(sequence, data_level)}
    GS --> G1{data_level == "DL1AB"?}
    G1 -->|yes| G1a[directory = options.directory / sequence.dl1_prod_id; glob dl1_*run*.h5]
    G1a --> GRET[return len(files)]
    G1 -->|no| G2{data_level == "DL2"?}
    G2 -->|yes| G2a[destination_dir(DL2,... sequence.dl2_prod_id); glob dl2_*run*.h5]
    G2a --> GRET
    G2 -->|no| G3{data_level == "DATACHECK"?}
    G3 -->|yes| G3a[glob datacheck in options.directory and destination_dir(DATACHECK,...)]
    G3a --> GRET
    G3 -->|no| G4[prefix/suffix from cfg; glob in options.directory]
    G4 --> GRET

    %% report_sequences / output_matrix
    Y --> RS{report_sequences}
    RS --> RS1[build header]
    RS1 --> RS2{tel_id in ["LST1","LST2"]?}
    RS2 --> RS3[extend header with DL1%, MUONS%, CAT-B, DL1AB%, DATACHECK%, DL2%]
    RS3 --> RS4[build matrix rows per sequence]
    RS4 --> RS5[padding = cfg.get("OUTPUT","PADDING")]
    RS5 --> RS6[output_matrix(matrix, padding)]

    RS6 --> OM{output_matrix}
    OM --> OM1[compute max_field_length per column]
    OM1 --> OM2[format each row with padding]
    OM2 --> OM3[log.info(stringrow)]

    %% is_sequencer_running
    Q1 --> ISR{is_sequencer_running(date)}
    ISR --> ISR1[summary_table = run_summary_table(date)]
    ISR1 --> ISR2[sacct_info = get_sacct_output(run_sacct())]
    ISR2 --> ISR3[for each run_id: filter JobName == f"LST1_{run:05d}"]
    ISR3 --> ISR4{any State RUNNING or PENDING?}
    ISR4 -->|yes| ISR4a[return True]
    ISR4 -->|no| ISR4b[return False]

    %% is_sequencer_completed
    Q2 --> ISC{is_sequencer_completed(date)}
    ISC --> ISC1[summary_table = run_summary_table(date)]
    ISC1 --> ISC2[data_runs = summary_table[run_type=="DATA"]]
    ISC2 --> ISC3[run_list = extract_runs(data_runs)]
    ISC3 --> ISC4[sequence_list = extract_sequences(options.date, run_list)]
    ISC4 --> ISC5{are_all_jobs_correctly_finished(sequence_list)?}
    ISC5 -->|yes| ISC5a[return True]
    ISC5 -->|no| ISC5b[log.info + return False]

    %% timeout_in_sequencer
    Q3 --> TO{timeout_in_sequencer(date)}
    TO --> TO1[summary_table = run_summary_table(date)]
    TO1 --> TO2[data_runs = summary_table[run_type=="DATA"]]
    TO2 --> TO3[sacct_info = get_sacct_output(run_sacct())]
    TO3 --> TO4[for each run_id: filter JobName]
    TO4 --> TO5{multiple JobID?}
    TO5 -->|yes| TO5a[keep last JobID]
    TO5 -->|no| TO6
    TO5a --> TO6{any State == TIMEOUT?}
    TO6 -->|yes| TO6a[return True]
    TO6 -->|no| TO6b[return False]
```
