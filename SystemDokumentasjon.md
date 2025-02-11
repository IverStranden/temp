<summary>
  <h2>Krav</h2>
</summary>
  
  - Flervalgsspørsmål med umiddelbar tilbakemelding på om svaret er riktig eller feil.
  - Poengsum som blir beregnet med slutten av quizen.
  - Gjenstartfunksjon for å la brukeren starte quizen på nytt med et enkelt klikk.
  - Design som møter grunnleggende krav til universell utforming og tilgjengelighet.
</details>

<details open>
<summary>
  <h2>Teknologier</h2>
</summary>
  
  - Vue.js
  - Appframe (CTP)
  - Bootstrap
  - Github
  - DrawSQL
  - Figma
  
</details>

<details open>
<summary>
  <h2>Arkitektur</h2>
</summary>
<ul>
    <li>
      <details>
        <summary>
          <h4>Tabeller</h4>:
        </summary>      
          <img src="https://github.com/user-attachments/assets/bac7d376-29b9-4933-84a7-12fe84c025fc" />
      </details>
    </li>
    <li>
      <details>
        <summary>
          <h4>Views</h4> 
        </summary>
        <table>
          <tr>
            <th>Navn</th>
            <th>Beskrivelse</th>
            <th>Kode</th>
          </tr>
          <tr>
            <td><b>aviw_IverFagproeve_Quiz_Leaderboard</b></td>
            <td>
              Dette viewet viser mengden rette svar, tid brukt, navn og rangering, i tillegg til session_ID (hvilket forsøk det var) og Quiz_ID.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    WITH CTE AS (
        SELECT 
            R.Session_ID, R.CreatedBy_ID, Q.ID AS Quiz_ID, 
            COUNT(DISTINCT R.Option_ID) AS AnsweredQuestions,
            (SELECT COUNT(*) 
             FROM dbo.atbl_IverFagproeve_Quiz_Questions AS QQ 
                WHERE QQ.Quiz_ID = Q.ID) AS TotalQuestions,
            COUNT(DISTINCT CASE WHEN A.Option_ID IS NOT NULL THEN R.Option_ID END) AS TotalCorrect,
            MIN(R.TimeSubmitted) AS StartTime, 
            MAX(R.TimeSubmitted) AS EndTime,
            DATEDIFF_BIG(MILLISECOND, MIN(R.TimeSubmitted), MAX(R.TimeSubmitted)) AS TimeSpentMilliseconds
        FROM dbo.atbv_IverFagproeve_Quiz_Results AS R
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Options AS O
                ON R.Option_ID = O.ID
            LEFT OUTER JOIN dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS A
                ON O.ID = A.Option_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
                ON O.Question_ID = QQ.ID
            INNER JOIN dbo.atbl_IverFagproeve_Quizzes AS Q
                ON QQ.Quiz_ID = Q.ID
        GROUP BY R.Session_ID, R.CreatedBy_ID, Q.ID
    )
    SELECT 
        CTE.Quiz_ID, Q.Name AS QuizName, CTE.CreatedBy_ID, CTE.Session_ID,
        CTE.AnsweredQuestions, CTE.TotalQuestions, CTE.TotalCorrect, 
        CTE.StartTime, CTE.EndTime, 
        FORMAT(CTE.TimeSpentMilliseconds / 1000.0, 'N2') AS TimeSpent,
        P.Name AS PersonName,
        RANK() OVER (
            PARTITION BY CTE.Quiz_ID 
            ORDER BY CTE.TotalCorrect DESC, CTE.TimeSpentMilliseconds ASC
        ) AS [Rank]
    FROM CTE AS CTE
        INNER JOIN dbo.atbv_IverFagproeve_Quizzes AS Q
            ON CTE.Quiz_ID = Q.ID
        INNER JOIN dbo.stbl_System_Persons AS P
            ON P.ID = CTE.CreatedBy_ID
        WHERE CTE.AnsweredQuestions = CTE.TotalQuestions
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>aviw_IverFagproeve_Quizzes_WithAverage</b></td>
            <td>
              Dette viewet viser alle relevante felter fra quiz tabellen, men har også med average compeltion tiden. Jeg valgte å ha 2 forskjellige views fordi dette viewet krevde en CTE.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    WITH SessionTimes AS (
        SELECT 
            QR.SESSION_ID,
            QQ.Quiz_ID,
            MIN(QR.TimeSubmitted) AS StartTime,
            MAX(QR.TimeSubmitted) AS EndTime
        FROM dbo.atbl_IverFagproeve_Quiz_Results AS QR
        INNER JOIN dbo.atbl_IverFagproeve_Quiz_Options AS QO
            ON QR.Option_ID = QO.ID
        INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
            ON QO.Question_ID = QQ.ID
        GROUP BY QR.SESSION_ID, QQ.Quiz_ID
    ),
    AvgCompletionTimes AS (
        SELECT 
            Quiz_ID,
            AVG(DATEDIFF(SECOND, StartTime, EndTime)) AS AvgCompletionTimeSeconds
        FROM SessionTimes
        GROUP BY Quiz_ID
    )
    SELECT 
        Q.PrimKey, Q.ID, Q.Created, Q.CreatedBy_ID, Q.Updated,
        Q.UpdatedBy_ID, Q.PassMinimum, Q.IsRestricted, Q.Description,
        Q.RevealAnswersImmediately, Q.TimeLimit, Q.Name, Q.OrgUnit_ID,
        OU.Name AS OrgUnitName, F.FileRef, F.FileSize, F.FileUpdated, F.Extension, F.FileName,
        F.PrimKey AS FilePrimKey, P.Name AS CreatedBy, Q.EnableHighContrast,
        ACT.AvgCompletionTimeSeconds,
        (SELECT COUNT(*) FROM dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
            WHERE QQ.Quiz_ID = Q.ID) AS QuestionCount
            FROM dbo.atbv_IverFagproeve_Quizzes AS Q
                LEFT OUTER JOIN dbo.stbl_System_OrgUnits AS OU
                    ON OU.ID = Q.OrgUnit_ID
                LEFT OUTER JOIN AvgCompletionTimes AS ACT
                    ON Q.ID = ACT.Quiz_ID
                LEFT OUTER JOIN dbo.atbl_IverFagproeve_Quiz_Files AS F
                    ON F.ID = Q.[File_ID]
                INNER JOIN dbo.stbl_System_Persons AS P
                    ON P.ID = Q.CreatedBy_ID
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>aviw_IverFagproeve_Quiz_Questions</b></td>
            <td>
              Dette viewet viser alle relevante felter fra questions tabellen, men har også et felt som viser om spørsmålene har alternativer festet til seg og felter som trengs for display av filer.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    SELECT
      Q.PrimKey, Q.ID, Q.Question, Q.Quiz_ID, Q.TimeLimit, Q.QuestionType_ID, Q.[Order], Q.[File_ID],
      QT.Name AS QuestionTypeName, F.FileRef, F.FileSize, F.FileUpdated, F.Extension, F.FileName,
      F.PrimKey AS FilePrimKey, Q.EnableHighContrast,
      CASE
          WHEN EXISTS (SELECT 1 FROM dbo.atbl_IverFagproeve_Quiz_Options AS O
                          WHERE O.Question_ID = Q.ID)
          THEN CAST(1 AS BIT)
          ELSE CAST(0 AS BIT)
      END AS HasOptions
      FROM dbo.atbv_IverFagproeve_Quiz_Questions AS Q
          INNER JOIN dbo.atbl_IverFagproeve_Quiz_QuestionTypes AS QT
              ON QT.ID = Q.QuestionType_ID
          LEFT OUTER JOIN dbo.atbl_IverFagproeve_Quiz_Files AS F
              ON F.ID = Q.[File_ID]
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>aviw_IverFagproeve_Quizzes</b></td>
            <td>
              Dette viewet viser alle relevante felter fra quiz tabellen, men har også et felter som trengs for display av filer og en som viser hvor mange spørsmål som er festet på.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    SELECT
      Q.PrimKey, Q.ID, Q.Created, Q.CreatedBy_ID, Q.Updated,
      Q.UpdatedBy_ID, Q.PassMinimum, Q.IsRestricted, Q.Description,
      Q.RevealAnswersImmediately, Q.TimeLimit, Q.Name, Q.OrgUnit_ID, Q.[File_ID], Q.EnableHighContrast,
      OU.Name AS OrgUnitName, F.FileRef, F.FileSize, F.FileUpdated, F.Extension, F.FileName,
      F.PrimKey AS FilePrimKey,
      (SELECT COUNT(*) FROM dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
          WHERE QQ.Quiz_ID = Q.ID) AS QuestionCount
      FROM dbo.atbv_IverFagproeve_Quizzes AS Q
          LEFT OUTER JOIN dbo.stbl_System_OrgUnits AS OU
              ON OU.ID = Q.OrgUnit_ID
          LEFT OUTER JOIN dbo.atbl_IverFagproeve_Quiz_Files AS F
              ON F.ID = Q.[File_ID]
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>aviw_IverFagproeve_Quiz_OptionsWithCorrect</b></td>
            <td>
              Dette viewet viser Options, og har ett felt som viser om de er det rette svaret. Dette viewet blir kun brukt i creator appen.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    SELECT
      O.PrimKey, O.ID, O.Question_ID, O.Content, OA.PrimKey AS AnswerPrimKey, OA.ID AS Answer_ID, OA.Deleted AS AnswerDeleted,
          CAST((SELECT 1 FROM dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS OA
              WHERE OA.Option_ID = O.ID) AS BIT) AS isCorrect
          FROM dbo.atbv_IverFagproeve_Quiz_Options AS O
              LEFT OUTER JOIN dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS OA
                  ON OA.Option_ID = O.ID
              </code></pre>
              </details>
            </td>
          </tr>
        </table>
      </details>
    </li>
    <li>
      <details>
        <summary>
          <h4>Stored Procedures</h4> 
        </summary>
        <table>
          <tr>
            <th>Navn</th>
            <th>Beskrivelse</th>
            <th>Kode</th>
          </tr>
          <tr>
            <td><b>astp_IverFagproeve_Quiz_DeleteQuiz</b></td>
            <td>
              Denne stored proceduren sletter quizen man spesifiserer og alt som er festa til den.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    (
        @Quiz_ID INT
    )
    AS
    BEGIN
        DELETE OA 
        FROM dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS OA
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Options AS O
                ON O.ID = OA.Option_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
                ON QQ.ID = O.Question_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quizzes AS Q
                ON Q.ID = QQ.Quiz_ID
            WHERE Q.ID = @Quiz_ID
        DELETE O 
        FROM dbo.atbl_IverFagproeve_Quiz_Options AS O
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
                ON QQ.ID = O.Question_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quizzes AS Q
                ON Q.ID = QQ.Quiz_ID
            WHERE Q.ID = @Quiz_ID
        DELETE QQ
            FROM dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
                INNER JOIN dbo.atbl_IverFagproeve_Quizzes AS Q
                    ON Q.ID = QQ.Quiz_ID
                WHERE Q.ID = @Quiz_ID
        DELETE FROM dbo.atbl_IverFagproeve_Quizzes
            WHERE ID = @Quiz_ID
    END
    SELECT 1
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>astp_IverFagproeve_Quiz_CalculateResults</b></td>
            <td>
              Denne stored proceduren regner ut hvor mange rette svar du fikk og hvor mange rette svar det var i quizen.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    (
        @Session_ID UNIQUEIDENTIFIER,
        @Quiz_ID INT
    )
    AS
    SELECT COUNT(*) AS MaxCorrect
        FROM dbo.atbl_IverFagproeve_Quiz_Questions AS QQ
            INNER JOIN dbo.atbl_IverFagproeve_Quizzes AS Q
                ON Q.ID = QQ.Quiz_ID
            WHERE Q.ID = @Quiz_ID
            GROUP BY QQ.Quiz_ID
    SELECT COUNT(*) AS CorrectAmount
        FROM dbo.atbv_IverFagproeve_Quiz_Results AS R
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Options AS O
                ON O.ID = R.Option_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS OA
                ON O.ID = OA.Option_ID
            WHERE R.Session_ID = @Session_ID
              </code></pre>
              </details>
            </td>
          </tr>
          <tr>
            <td><b>astp_IverFagproeve_Quiz_DeleteQuestion</b></td>
            <td>
              Denne stored proceduren sletter spørsmålet man spesifiserer og alt som er festa under den.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>
              </summary> 
              <pre><code>
    (
        @QuestionPrimKey UNIQUEIDENTIFIER
    )
    AS
    BEGIN
        DELETE OA 
        FROM dbo.atbl_IverFagproeve_Quiz_OptionsAnswers AS OA
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Options AS O
                ON O.ID = OA.Option_ID
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS Q
                ON Q.ID = O.Question_ID
            WHERE Q.PrimKey = @QuestionPrimKey
        DELETE O 
        FROM dbo.atbl_IverFagproeve_Quiz_Options AS O
            INNER JOIN dbo.atbl_IverFagproeve_Quiz_Questions AS Q
                ON Q.ID = O.Question_ID
            WHERE Q.PrimKey = @QuestionPrimKey
        DELETE FROM dbo.atbl_IverFagproeve_Quiz_Questions 
            WHERE PrimKey = @QuestionPrimKey
    END
    SELECT 1
              </code></pre>
              </details>
            </td>
          </tr>
        </table>
      </details>
    </li>
  </ul>

</details>

<details open>
<summary>
  <h2>Sikkerhet</h2>
</summary>

I Appframe håndteres autentisering på en sikker måte. Brukerne kan velge å logge inn med Microsoft-konto eller via SQL-login med telefonnummer eller e-postadresse og passord.
Om Microsoft-login velges, skjer autentiseringen gjennom Microsofts sine systemer, som returnerer en autentiseringstoken som kobles til brukeren i Appframe. Ved SQL-login sjekkes brukernavn og passord mot databasen via et API-kall, som gjør at innloggingen blir behandlet på en trygg måte.
For å gjøre sikkerheten bedre så er det også støtte for 2FA (to-faktor-autentisering), som gir et ekstra ledd med beskyttelse. I tillegg benyttes rollebasert tilgangsstyring for å definere hva enhver bruker kan gjøre i systemet.
Man tildeler roller til brukere som gir tilgang til spesifikke moduler, apper og tabeller.

</details>

<details open>
<summary>
  <h2>Grensesnittbeskrivelse</h2>
</summary>
    - Brukerveiledning
</details>

<details open>
<summary>
  <h2>Hindringer under utviklingen</h2>
</summary>
  
  - Vue.js
  - Appframe (CTP)
  - Bootstrap
  - Github
  - DrawSQL
  - Figma

</details>

<details open>
<summary>
  <h2>Avvik fra plan</h2>
</summary>
  
  - Vue.js
  - Appframe (CTP)
  - Bootstrap
  - Github
  - DrawSQL
  - Figma

</details>
