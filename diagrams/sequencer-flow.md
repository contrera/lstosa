flowchart TD
  %% src/osa/scripts/sequencer.py (commit 9f491ae3c...)
  A0["__main__"] -->|python executes| A1["main()"]

  subgraph MAIN["main()"]
    M1["sequencer_cli_parsing()"] --> M2{"options.verbose?"}
    M2 -->|yes| M3["log.setLevel(DEBUG)"]
    M2 -->|no| M4["log.setLevel(INFO)"]
    M3 --> M5["tag = gettag()"]
    M4 --> M5
    M5 --> M6["start(tag)"]
    M6 --> M7{"options.tel_id in ['LST1','LST2']?"}
    M7 -->|yes| M8["single_process(options.tel_id)"]
    M7 -->|no| M9["log.error('Process mode not supported yet')"]
  end

  subgraph SP["single_process(telescope)"]
    S1["database = cfg.get('database','path')"] --> S2{"database truthy?"}
    S2 -->|yes| S3["osadb.start_processing(date_to_iso(options.date))"]
    S2 -->|no| S4["(skip db start)"]
    S3 --> S5["Init sequence_list=[]; set options.tel_id=telescope"]
    S4 --> S5
    S5 --> S6["options.directory = analysis_path(options.tel_id)\noptions.log_directory = options.directory/'log'"]
    S6 --> S7{"not options.simulate?"}
    S7 -->|yes| S8["os.makedirs(options.log_directory, exist_ok=True)"]
    S7 -->|no| S9["(no log dir creation)"]
    S8 --> S10["summary_table = run_summary_table(options.date)"]
    S9 --> S10
    S10 --> S11{"len(summary_table)==0?"}
    S11 -->|yes| S12["log.warning(...)\nsys.exit(0)"]
    S11 -->|no| S13{"(not options.no_gainsel)\nAND (not GainSel_finished(options.date))?"}
    S13 -->|yes| S14["log.info(...)\nsys.exit()"]
    S13 -->|no| S15{"is_day_closed()?"}
    S15 -->|yes| S16["log.info(...)\nreturn sequence_list"]
    S15 -->|no| S17{"(not options.test)\nAND (not options.simulate)?"}

    S17 -->|no| S24["sequence_list = build_sequences(options.date)"]
    S17 -->|yes| S18{"is_sequencer_running(options.date)?"}
    S18 -->|yes| S19["log.info(...)\nsys.exit(0)"]
    S18 -->|no| S20{"is_sequencer_completed(options.date)\nAND (not options.force_submit)?"}
    S20 -->|yes| S21["log.info(...)\nsys.exit(0)"]
    S20 -->|no| S22{"timeout_in_sequencer(options.date)\nAND (not options.force_submit)?"}
    S22 -->|yes| S23["log.info(...)\nsys.exit(0)"]
    S22 -->|no| S24

    S24 --> S25["prepare_jobs(sequence_list)"]
    S25 --> S26["update_job_info(sequence_list)"]
    S26 --> S27["get_veto_list(sequence_list)"]
    S27 --> S28["get_closed_list(sequence_list)"]
    S28 --> S29["update_sequence_status(sequence_list)"]
    S29 --> S30{"not options.no_submit?"}
    S30 -->|yes| S31["submit_jobs(sequence_list)"]
    S30 -->|no| S32["(skip submit)"]
    S31 --> S33["report_sequences(sequence_list)"]
    S32 --> S33
    S33 --> S34["return sequence_list"]
  end

  subgraph UJI["update_job_info(sequence_list)"]
    J1{"options.test?"}
    J1 -->|yes| J2["return"]
    J1 -->|no| J3["sacct_output = run_sacct()\nsqueue_output = run_squeue()"]
    J3 --> J4["set_queue_values(\n  sacct_info=get_sacct_output(sacct_output),\n  squeue_info=get_squeue_output(squeue_output),\n  sequence_list=sequence_list\n)"]
  end

  subgraph USS["update_sequence_status(seq_list)"]
    U1["for seq in seq_list"] --> U2{"seq.type == 'PEDCALIB'?"}
    U2 -->|yes| U3["seq.calibstatus = int(Decimal(get_status_for_sequence(seq,'CALIB')*100)/seq.subruns)"]
    U2 -->|no| U4{"seq.type == 'DATA'?"}
    U4 -->|yes| U5["Compute statuses via get_status_for_sequence:\nDL1, DL1AB, DATACHECK, MUON, DL2\nand seq.catbstatus = check_catB_status(seq)"]
    U4 -->|no| U6["(no updates for other types)"]
  end

  subgraph CATB["check_catB_status(seq)"]
    C1["catbstatus='None'"] --> C2{"seq.type == 'DATA'?"}
    C2 -->|no| C9["return catbstatus"]
    C2 -->|yes| C3["closed_files = options.directory.glob('catB*{run}*.closed')"]
    C3 --> C4{"closed_files found?"}
    C4 -->|yes| C5["catbstatus='CLOSED'"]
    C4 -->|no| C6["log_files = options.log_directory.glob('catB_calibration_{run}_*.err')"]
    C6 --> C7{"log_files found?"}
    C7 -->|no| C9
    C7 -->|yes| C8["job_id = parse from latest err filename\nsacct_info = get_sacct_output(run_sacct(job_id))\nif not empty: catbstatus = sacct_info.iloc[0]['State']"]
    C5 --> C9
    C8 --> C9
  end

  subgraph GSFS["get_status_for_sequence(sequence, data_level)"]
    G1{"data_level == 'DL1AB'?"}
    G1 -->|yes| G2["try: directory=options.directory/sequence.dl1_prod_id\nfiles=glob('dl1_LST-1*{run}*.h5')\nexcept AttributeError: return 0"]
    G1 -->|no| G3{"data_level == 'DL2'?"}
    G3 -->|yes| G4["try: directory=destination_dir('DL2', create_dir=False, dl2_prod_id=sequence.dl2_prod_id)\nfiles=glob('dl2_LST-1*{run}*.h5')\nexcept AttributeError: return 0"]
    G3 -->|no| G5{"data_level == 'DATACHECK'?"}
    G5 -->|yes| G6["try: directory=options.directory/sequence.dl1_prod_id\nalt=destination_dir('DATACHECK', create_dir=False, dl1_prod_id=sequence.dl1_prod_id)\nfiles=glob('datacheck_dl1_LST-1*{run}*.h5') + alt.glob(...)\nexcept AttributeError: return 0"]
    G5 -->|no| G7["prefix=cfg.get('PATTERN', f'{data_level}PREFIX')\nsuffix=cfg.get('PATTERN', f'{data_level}SUFFIX')\nfiles=options.directory.glob(f'{prefix}*{run}*{suffix}')"]
    G2 --> G8["return len(files)"]
    G4 --> G8
    G6 --> G8
    G7 --> G8
  end

  subgraph RS["report_sequences(sequence_list)"]
    R1["Build header"] --> R2{"options.tel_id in ['LST1','LST2']?"}
    R2 -->|yes| R3["header += ['DL1%','MUONS%','CAT-B','DL1AB%','DATACHECK%','DL2%']"]
    R2 -->|no| R4["(header unchanged)"]
    R3 --> R5["matrix=[header]"]
    R4 --> R5
    R5 --> R6["for sequence in sequence_list: build row_list (core columns)"]
    R6 --> R7{"sequence.type in ['DRS4','PEDCALIB']?"}
    R7 -->|yes| R8["row_list += (None x6)"]
    R7 -->|no| R9{"sequence.type == 'DATA'?"}
    R9 -->|yes| R10["row_list += (dl1status, muonstatus, catbstatus, dl1abstatus, datacheckstatus, dl2status)"]
    R9 -->|no| R11["(no extra cols)"]
    R8 --> R12["matrix.append(row_list)"]
    R10 --> R12
    R11 --> R12
    R12 --> R13["padding=int(cfg.get('OUTPUT','PADDING'))"]
    R13 --> R14["output_matrix(matrix, padding)"]
  end

  subgraph OM["output_matrix(matrix, padding_space)"]
    O1["Compute max_field_length per column"] --> O2["For each row: build padded stringrow"] --> O3["log.info(stringrow)"]
  end

  subgraph ISR["is_sequencer_running(date)"]
    IR1["summary_table = run_summary_table(date)\nsacct_info = get_sacct_output(run_sacct())"] --> IR2["for run in summary_table['run_id']"]
    IR2 --> IR3["jobs_run = sacct_info[JobName == f'LST1_{run:05d}']"]
    IR3 --> IR4["queued_jobs = State in {RUNNING,PENDING}"]
    IR4 --> IR5{"len(queued_jobs) != 0?"}
    IR5 -->|yes| IR6["return True"]
    IR5 -->|no| IR7["continue"]
    IR7 --> IR8["return False"]
  end

  subgraph ISC["is_sequencer_completed(date)"]
    IC1["summary_table = run_summary_table(date)\ndata_runs = summary_table[run_type=='DATA']"] --> IC2["run_list=extract_runs(data_runs)"]
    IC2 --> IC3["sequence_list=extract_sequences(options.date, run_list)"]
    IC3 --> IC4{"are_all_jobs_correctly_finished(sequence_list)?"}
    IC4 -->|yes| IC5["return True"]
    IC4 -->|no| IC6["log.info(...)\nreturn False"]
  end

  subgraph TOS["timeout_in_sequencer(date)"]
    TO1["summary_table = run_summary_table(date)\ndata_runs = summary_table[run_type=='DATA']\nsacct_info=get_sacct_output(run_sacct())"] --> TO2["for run in data_runs['run_id']"]
    TO2 --> TO3["jobs_run = sacct_info[JobName==f'LST1_{run:05d}']"]
    TO3 --> TO4{"multiple JobID.unique()?"}
    TO4 -->|yes| TO5["last_job_id = max JobID\njobs_run = sacct_info[JobID==last_job_id]"]
    TO4 -->|no| TO6["(keep jobs_run)"]
    TO5 --> TO7["timeout_jobs = State=='TIMEOUT'"]
    TO6 --> TO7
    TO7 --> TO8{"len(timeout_jobs)!=0?"}
    TO8 -->|yes| TO9["return True"]
    TO8 -->|no| TO10["continue"]
    TO10 --> TO11["return False"]
  end
