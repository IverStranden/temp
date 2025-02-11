<details open>
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
                <h4>Expand</h4>:
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
                <h4>Expand</h4>:
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
              Dette viewet viser alle relevante felter fra questions tabellen, men har også et felt som viser om spørsmålene har alternativer festet til seg.
            </td>
            <td>
              <details>
              <summary>
                <h4>Expand</h4>:
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
        </table>
      </details>
    </li>
  </ul>

</details>

<details open>
<summary>
  <h2>Sikkerhet</h2>
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
  <h2>Grensesnittbeskrivelse</h2>
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
