# Project-2
1. Without any condition :

def schedule_machines(JOBS, MACHINES):

    # Create the mip solver with the SCIP backend.
    m = pywraplp.Solver.CreateSolver('SCIP')

    #Big M
    tmax = max([JOBS[job_id]['release'] for job_id in JOBS.keys()]) + sum([JOBS[job_id]['duration'] for job_id in JOBS.keys()])

    # decision variables
    completion_time = []
    auxilaryVar = []
    for job in range(len(JOBS.keys())):
        suffix = '_%s' % job
        completion_time.append(m.NumVar(0, tmax, 'complete' + suffix))

        auxilaryVar_nested=[]
        for job_k in JOBS.keys():
            auxilaryVar_nested.append(m.BoolVar('y' + suffix))
        auxilaryVar.append(auxilaryVar_nested)
    
    # additional decision variables for use in the objecive
    makespan   = m.NumVar(0, tmax, 'makespan')
    # for binary assignment of jobs to machines
    assignJob2Mch = []
    for job_id in JOBS.keys():
        assignJob2Mch.append([])
        for machine_id in MACHINES:
            assignJob2Mch[-1].append(m.BoolVar('assignJob2Mch_' + job_id + '_' + machine_id))
    # constraints 

    # Each job assigned to exactly one machine
    for job_id in JOBS.keys():
    m.Add(sum(assignJob2Mch[job_id][m] for m in range(len(MACHINES))) == 1)

    # Completion time >= job duration if assigned to machine
    for job_id in JOBS.keys():
        job = JOBS[job_id]
        for m in range(len(MACHINES)):
            machine_id = MACHINES[m]
        m.Add(completion_time[job_id] >= job['duration'] * assignJob2Mch[job_id][m])

    # Completion time <= makespan for objective calculation
    for job_id in JOBS.keys():
        m.Add(completion_time[job_id] <= makespan)

    # Additional constraints to model disjunctive relationships between jobs
    for j, job_id in enumerate(JOBS.keys()):
        job = JOBS[job_id]
        for k, job_id_k in enumerate(JOBS.keys()):
            if k != j:
               m.Add(completion_time[j] + job['duration'] <= completion_time[k] + tmax * auxilaryVar[j][k])
               m.Add(completion_time[k] + JOBS[job_id]['duration'] <= completion_time[j] + tmax * (1 - auxilaryVar[j][k]))
    

    # objective function
    m.Minimize(makespan)
    
    status = m.Solve()

    if status == pywraplp.Solver.OPTIMAL:
    
        SCHEDULE = {}
        for j,job_id in enumerate(JOBS.keys()):
            mm = [mach for mach in range(len(MACHINES)) if assignJob2Mch[j][mach].solution_value()==1]
            SCHEDULE[job_id] = {'machine': MACHINES[mm[0]], 
                                'start': completion_time[j].solution_value() - JOBS[job_id]['duration'], 
                                'finish': completion_time[j].solution_value() }

        print(SCHEDULE)
        print(m.Objective().Value())
        return SCHEDULE

    else:
        print('The problem does not have an optimal solution.')

SCHEDULE = opt_schedule(JOBS)
gantt(JOBS, SCHEDULE)
kpi(JOBS, SCHEDULE)



2. With release time 

def schedule_machines(JOBS, MACHINES):

    # Create the mip solver with the SCIP backend.
    m = pywraplp.Solver.CreateSolver('SCIP')

    # Big M
    tmax = max([JOBS[job_id]['release'] for job_id in JOBS.keys()]) + sum([JOBS[job_id]['duration'] for job_id in JOBS.keys()])

    # Decision variables
    completion_time = []
    start_time = []
    auxilaryVar = []
    for job in range(len(JOBS.keys())):
        suffix = '_%s' % job
        completion_time.append(m.NumVar(0, tmax, 'complete' + suffix))
        start_time.append(m.NumVar(0, tmax, 'start' + suffix))

        auxilaryVar_nested = []
        for job_k in JOBS.keys():
            auxilaryVar_nested.append(m.BoolVar('y' + suffix))
        auxilaryVar.append(auxilaryVar_nested)

    # Additional decision variables for use in the objective
    makespan = m.NumVar(0, tmax, 'makespan')

    # For binary assignment of jobs to machines
    assignJob2Mch = []
    for job_id in JOBS.keys():
        assignJob2Mch.append([])
        for machine_id in MACHINES:
            assignJob2Mch[-1].append(m.BoolVar('assignJob2Mch_' + job_id + '_' + machine_id))

    # Constraints
    for j, job_id in enumerate(JOBS.keys()):
        job = JOBS[job_id]
        for m in range(len(MACHINES)):
            machine_id = MACHINES[m]
            m.Add(completion_time[j] >= job['duration'] * assignJob2Mch[j][m])  # Completion time >= job duration if assigned to machine
            m.Add(start_time[j] >= job['release'])  # Start time >= job release time
            m.Add(completion_time[j] <= makespan)  # Completion time <= makespan for objective calculation

        for k, job_id_k in enumerate(JOBS.keys()):
            if k != j:
                m.Add(completion_time[j] + JOBS[job_id_k]['duration'] <= completion_time[k] + tmax * auxilaryVar[j][k])
                m.Add(completion_time[k] + JOBS[job_id]['duration'] <= completion_time[j] + tmax * (1 - auxilaryVar[j][k]))

        # Adjust start time based on arrival time
        m.Add(start_time[j] >= job['arrival'])
        m.Add(start_time[j] <= job['arrival'] + job['duration'])

    # Objective function
    m.Minimize(makespan)

    status = m.Solve()

    if status == pywraplp.Solver.OPTIMAL:

        SCHEDULE = {}
        for j, job_id in enumerate(JOBS.keys()):
            mm = [mach for mach in range(len(MACHINES)) if assignJob2Mch[j][mach].solution_value() == 1]
            SCHEDULE[job_id] = {'machine': MACHINES[mm[0]],
                                'start': start_time[j].solution_value(),
                                'finish': completion_time[j].solution_value()}

        print(SCHEDULE)
        print(m.Objective().Value())
        return SCHEDULE

    else:
        print('The problem does not have an optimal solution.')
        
3. Considering priorities 


  
