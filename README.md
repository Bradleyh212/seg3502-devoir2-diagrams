# SEG3502 — Devoir 2 : diagrammes UML

Diagrammes de séquence des cas d’utilisation **RegisterDepartmentEmployee** (Question 1) et **SelectWorkShifts** (Question 2).

Les diagrammes sont affichés ci-dessous. Les fichiers `.mmd` restent disponibles pour modification dans Mermaid Live ou un autre éditeur compatible.

Propositions de travail : les hypothèses de conception figurent dans les notes des diagrammes.

## Sommaire

- [Question 1 — Niveau 1](q1-level1.mmd)

- [Question 1 — Niveau 2 : registerEmployee](q1-level2.mmd)

- [Question 1 — Niveau 2 : assignEmployeeToDepartment](q1-level2-assign.mmd)

- [2.1 — Question 2 : niveau 1](#21--question-2--niveau-1)

- [2.2 — Question 2 : niveau 2](#22--question-2--niveau-2)


## Question 1 — Niveau 1


[Source Mermaid](q1-level1.mmd)


```mermaid
sequenceDiagram
    autonumber
    actor Client as Directeur via interface
    participant App as RegisterDepartmentEmployee
    participant EF as EmployeeFacade (proposee)
    participant DF as DepartmentFacade (proposee)

    Note over App,DF: Architecture validee - operations proposees.<br/>Scenario de succes des deux commandes, sans Scheduling.
    Client->>App: registerEmployee(empInfo: EmployeeRegistrationDto)
    Note over App,DF: Preconditions : Airport Manager connecte, SIN inexistant.<br/>Verification detaillee au niveau 2.
    App->>EF: registerEmployee(empInfo)
    EF-->>App: empId: EmployeeId
    App-->>Client: empId
    Note over Client,DF: empId est retourne avant le choix de deptId.<br/>La consultation des departements est hors des deux commandes.

    Client->>App: assignEmployeeToDepartment(empId: EmployeeId, deptId: DepartmentId)
    App->>EF: ensureAssignable(empId)
    EF-->>App: employe existant et sans departement
    App->>DF: ensureExists(deptId)
    DF-->>App: departement existant
    Note over App,DF: Poursuivre seulement si les trois preconditions sont satisfaites.<br/>Coordination des mises a jour a preciser au niveau 2.
    App->>EF: assignDepartment(empId, deptId)
    EF-->>App: Employee.department = deptId
    App->>DF: addEmployee(deptId, empId)
    DF-->>App: empId appartient a Department.employees
    App-->>Client: affectation reussie
    Note over EF,DF: References entre agregats par identifiant.<br/>Emissions des evenements detaillees au niveau 2.
```


## Question 1 — Niveau 2 : registerEmployee


[Source Mermaid](q1-level2.mmd)


```mermaid
sequenceDiagram
    autonumber
    participant App as RegisterDepartmentEmployee
    box Sous-domaine Employee
        participant EF as EmployeeFacade
        participant ER as EmployeeRepository
        participant F as EmployeeFactory
        participant User as utilisateurConnecte: Employee
    end

    Note over App,F: NIVEAU 2 - registerEmployee(empInfo)

    App->>EF: registerEmployee(empInfo: EmployeeRegistrationDto)
    EF->>ER: findById(currentUserId)
    ER-->>EF: utilisateurConnecte
    EF->>User: getJobTitle()
    User-->>EF: jobTitle
    EF->>ER: existsBySin(empInfo.sin)
    ER-->>EF: false
    Note over EF,User: Preconditions : jobTitle.title = Airport Manager et SIN inexistant.<br/>Sinon, arret sans creation.

    EF->>F: create(empInfo)
    F->>ER: nextIdentity()
    ER-->>F: empId: EmployeeId neuf
    create participant Job as job: JobTitle
    F->>Job: new JobTitle(empInfo.jobTitle)
    create participant Addr as addr: Address
    F->>Addr: new Address(street, unitNumber [0..1], city, postalCode)
    create participant Bank as bank: BankInfo
    F->>Bank: new BankInfo(accountNumber, branchNumber, transitNumber)
    create participant Emp as emp: Employee (racine)
    F->>Emp: new Employee(empId, firstName, lastName, dateOfBirth, sin, job, addr, bank)
    Note over Emp: emp.jobTitle = job, emp.address = addr, emp.bankInfo = bank.<br/>Valeurs tirees de empInfo.
    Emp->>Emp: emettre evenement de creation de l'employe (empId)
    Note over Emp: Point d'emission de l'evenement de domaine.
    F-->>EF: emp

    EF->>ER: save(emp)
    EF-->>App: empId: EmployeeId
```


## Question 1 — Niveau 2 : assignEmployeeToDepartment


[Source Mermaid](q1-level2-assign.mmd)


```mermaid

sequenceDiagram
    autonumber
    participant App as RegisterDepartmentEmployee
    box Sous-domaine Employee
        participant EF as EmployeeFacade
        participant ER as EmployeeRepository
        participant Emp as emp: Employee (racine)
    end
    box Sous-domaine Department
        participant DF as DepartmentFacade
        participant DR as DepartmentRepository
        participant Dept as dept: Department (racine)
    end

    Note over App,Dept: NIVEAU 2 - assignEmployeeToDepartment(empId: EmployeeId, deptId: DepartmentId).<br/>Suite de q1-level2.mmd, apres retour d'empId et choix de deptId.<br/>Operations et depots proposes, non imposes par le PDF.
    rect rgba(128, 128, 128, 0.10)
        App->>EF: ensureAssignable(empId)
        EF->>ER: findById(empId)
        ER-->>EF: emp
        Note over EF,Emp: Si emp est absent, arreter la commande sans modification.
        EF->>Emp: getDepartment()
        Emp-->>EF: absent
        Note over EF,Emp: Si un department est deja defini, arreter sans modification.<br/>Aucune mutation dans ensureAssignable.
        EF-->>App: employe existant et sans departement

        App->>DF: ensureExists(deptId)
        DF->>DR: findById(deptId)
        DR-->>DF: dept
        Note over DF,Dept: Si dept est absent, arreter la commande sans modification.<br/>Aucune mutation dans ensureExists.
        DF-->>App: departement existant
    end

    Note over App,Dept: Hypothese proposee : une transaction coordonnee couvre verification et mises a jour.<br/>Isolation ou verrouillage adequat contre modification concurrente et annulation des deux sauvegardes en cas d'echec.<br/>Le PDF ne precise pas ce mecanisme, a valider selon les conventions du cours.
    rect rgba(128, 128, 128, 0.16)
        App->>EF: assignDepartment(empId, deptId)
        EF->>ER: findById(empId)
        ER-->>EF: emp
        Note over EF,Emp: Meme agregat dans le contexte transactionnel propose.<br/>La racine protege aussi la regle de non-affectation.
        EF->>Emp: assignDepartment(deptId)
        Emp->>Emp: verifier department absent puis department = deptId
        Emp->>Emp: emettre un evenement d'affectation de l'employe (empId, deptId)
        Note over Emp: Point d'emission propose apres la mutation.<br/>Description de l'evenement, pas un nom fourni par le PDF.<br/>Enregistrement local, sans publication externe avant succes global.
        Emp-->>EF: affectation effectuee
        EF->>ER: save(emp)
        ER-->>EF: sauvegarde dans la transaction
        EF-->>App: Employee.department = deptId
    end

    rect rgba(128, 128, 128, 0.10)
        App->>DF: addEmployee(deptId, empId)
        DF->>DR: findById(deptId)
        DR-->>DF: dept
        DF->>Dept: addEmployee(empId)
        Dept->>Dept: employees = employees union {empId}
        Dept->>Dept: emettre un evenement d'ajout au departement (deptId, empId)
        Note over Dept: Point d'emission propose apres la mutation.<br/>Description de l'evenement, pas un nom fourni par le PDF.<br/>Enregistrement local, sans publication externe avant succes global.
        Dept-->>DF: identifiant ajoute
        DF->>DR: save(dept)
        DR-->>DF: sauvegarde dans la transaction
        DF-->>App: empId appartient a Department.employees
    end

    Note over App,Dept: Succes de la commande seulement apres validation de la transaction.<br/>Hypothese : publication externe des evenements apres succes global.<br/>Aucun objet Employee transmis a Department, aucun objet Department transmis a Employee.<br/>Pas de fabrique ici : les deux agregats existent deja.
    Note over DF,Dept: La notification du responsable figure dans le scenario du cas d'utilisation,<br/>pas dans les postconditions de ces commandes. Le PDF ne fournit ni canal ni gestionnaire.<br/>Un traitement ulterieur des evenements est une possibilite a valider, non une notification realisee ici.

```


## 2.1 — Question 2 : niveau 1

[Source Mermaid](q2-level1.mmd)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Employe via interface
    participant App as SelectWorkShifts
    participant EF as EmployeeFacade (proposee)
    participant SF as SchedulingFacade (proposee)
    participant DF as DepartmentFacade (proposee)

    Note over App,DF: Architecture validee - operations proposees.<br/>Scenario de succes de la commande selectShifts.
    Note over Client,App: Etapes 1-2 (consultation des quarts) hors du contrat.<br/>Precondition du cas : employe authentifie.
    Client->>App: selectShifts(empId: EmployeeId, shiftIds: ShiftId[])

    App->>EF: getAssignedDepartment(empId)
    EF-->>App: deptId: DepartmentId
    Note over App,EF: Precondition : employe existant et affecte a un departement.

    App->>SF: ensureUpcoming(shiftIds)
    SF-->>App: chaque quart existe et est a venir

    App->>DF: ensureContainsShifts(deptId, shiftIds)
    DF-->>App: chaque quart appartient au departement de l'employe
    Note over App,DF: Poursuivre seulement si toutes les preconditions sont satisfaites.<br/>Verification detaillee au niveau 2.

    App->>EF: addPreferredShifts(empId, shiftIds)
    EF-->>App: shiftIds inclus dans Employee.preferedShifts
    App-->>Client: selection enregistree
    Note over EF,DF: References entre agregats par identifiant.<br/>Emission de l'evenement detaillee au niveau 2.
```


## 2.2 — Question 2 : niveau 2

[Source Mermaid](q2-level2.mmd)

```mermaid
sequenceDiagram
    autonumber
    participant App as SelectWorkShifts
    box Sous-domaine Employee
        participant EF as EmployeeFacade
        participant ER as EmployeeRepository
        participant Emp as emp: Employee (racine)
    end
    box Sous-domaine Scheduling
        participant SF as SchedulingFacade
        participant SR as ShiftRepository
        participant Sh as shift: Shift (racine)
    end
    box Sous-domaine Department
        participant DF as DepartmentFacade
        participant DR as DepartmentRepository
        participant Dept as dept: Department (racine)
    end

    Note over App,Dept: NIVEAU 2 - selectShifts(empId, shiftIds)

    App->>EF: getAssignedDepartment(empId)
    EF->>ER: findById(empId)
    ER-->>EF: emp
    EF->>Emp: getDepartment()
    Emp-->>EF: deptId
    EF-->>App: deptId

    App->>SF: ensureUpcoming(shiftIds)
    SF->>SR: findAllById(shiftIds)
    SR-->>SF: shifts
    loop pour chaque shift
        SF->>Sh: isUpcoming(today)
        Sh-->>SF: true
    end
    SF-->>App: quarts existants et a venir

    App->>DF: ensureContainsShifts(deptId, shiftIds)
    DF->>DR: findById(deptId)
    DR-->>DF: dept
    loop pour chaque shiftId
        DF->>Dept: containsShift(shiftId)
        Dept-->>DF: true
    end
    DF-->>App: quarts du departement

    Note over App,Dept: Si une verification echoue (employe absent ou sans departement,<br/>quart inexistant, passe ou hors departement), la commande s'arrete sans modification.

    App->>EF: addPreferredShifts(empId, shiftIds)
    EF->>ER: findById(empId)
    ER-->>EF: emp
    EF->>Emp: addPreferredShifts(shiftIds)
    Emp->>Emp: preferedShifts = preferedShifts union shiftIds
    Emp->>Emp: emettre evenement de selection des quarts (empId, shiftIds)
    Note over Emp: Point d'emission de l'evenement de domaine.
    Emp-->>EF: quarts ajoutes
    EF->>ER: save(emp)
    EF-->>App: preferences enregistrees
```
