# SEG3502 — Devoir 2 : diagrammes UML

Diagrammes de séquence du cas d’utilisation **RegisterDepartmentEmployee**.

Les diagrammes sont affichés ci-dessous. Les fichiers `.mmd` restent disponibles pour modification dans Mermaid Live ou un autre éditeur compatible.

Propositions de travail : les hypothèses de conception figurent dans les notes des diagrammes.

## Sommaire

- [Question 1 — Niveau 1](q1-level1.mmd)

- [Question 1 — Niveau 2 : registerEmployee](q1-level2.mmd)

- [Question 1 — Niveau 2 : assignEmployeeToDepartment](q1-level2-assign.mmd)


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
    Note over App,EF: Preconditions : utilisateur connecte Airport Manager et SIN inexistant.<br/>Verification detaillee au niveau 2.
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

    Note over App,F: NIVEAU 2 - registerEmployee.<br/>Suite de la Q1 dans q1-level2-assign.mmd.<br/>Noms des operations, fabrique et depot proposes, non imposes par le PDF.
    App->>EF: registerEmployee(empInfo: EmployeeRegistrationDto)
    rect rgba(128, 128, 128, 0.10)
        Note over EF,User: Preconditions - currentUserId provient du contexte authentifie.<br/>Il ne provient pas du DTO du nouvel employe.
        EF->>ER: findById(currentUserId)
        ER-->>EF: utilisateurConnecte
        EF->>User: getJobTitle()
        User-->>EF: jobTitle du demandeur
        Note over EF,User: Exiger un Employee existant et jobTitle.title = Airport Manager.<br/>Arreter la commande en cas d'echec, avant toute creation.
        EF->>ER: existsBySin(empInfo.sin)
        ER-->>EF: false
        Note over EF,ER: Si true, arreter la commande sans creation.<br/>Hypothese : le depot garantit aussi l'unicite du SIN a l'enregistrement.
    end

    rect rgba(128, 128, 128, 0.16)
        EF->>F: create(empInfo)
        F->>ER: nextIdentity()
        ER-->>F: empId: EmployeeId neuf (UUID)
        Note over F,ER: Allocation de l'identifiant par le depot = choix propose.<br/>La garantie requise est un nouvel identifiant Employee.
        create participant Job as job: JobTitle
        F->>Job: new JobTitle(empInfo.jobTitle)
        Note over Job: job.title = empInfo.jobTitle
        create participant Addr as addr: Address
        F->>Addr: new Address(empInfo.address.street, empInfo.address.unitNumber,<br/>empInfo.address.city, empInfo.address.postalCode)
        Note over Addr: street, unitNumber, city, postalCode initialises.<br/>unitNumber conserve son caractere optionnel [0..1].
        create participant Bank as bank: BankInfo
        F->>Bank: new BankInfo(empInfo.bankInfo.accountNumber,<br/>empInfo.bankInfo.branchNumber, empInfo.bankInfo.transitNumber)
        Note over Bank: accountNumber, branchNumber, transitNumber initialises.
        create participant Emp as emp: Employee (racine)
        F->>Emp: new Employee(empId, empInfo.firstName, empInfo.lastName,<br/>empInfo.dateOfBirth, empInfo.sin, job, addr, bank)
        Note over Emp: id, firstName, lastName, dateOfBirth, sin initialises.<br/>emp.jobTitle = job, emp.address = addr, emp.bankInfo = bank.<br/>Hypothese : emp.department absent avant affectation.<br/>Les trois objets valeur appartiennent a l'agregat Employee.
        Emp->>Emp: emettre un evenement de creation de l'employe (empId)
        Note over Emp: Point d'emission propose dans la racine apres initialisation.<br/>Description de l'evenement, pas un nom fourni par le PDF.<br/>Emission = enregistrement local dans les evenements de l'agregat.
        F-->>EF: emp
    end

    rect rgba(128, 128, 128, 0.10)
        EF->>ER: save(emp)
        ER-->>EF: enregistrement reussi
        Note over EF,Emp: save persiste l'agregat complet, objets valeur inclus.<br/>Hypothese : publication externe des evenements apres succes<br/>de la transaction, mecanisme non specifie et non modelise.
        EF-->>App: empId: EmployeeId
    end

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
