### Collections:


```bash

Collections:

ClearCollect(
    RelevantProgramGuidanceIDs,
    Filter(
        EmsProgramGuidance,
        ArtefactMetadataID.Id = DropdownArtefact.Selected.ID,
        Program.Id = DropdownProgram.Selected.ID
    ).ID
);


ClearCollect(
    RelevantGuidanceItemJoins,
    Filter(
        EmsProgramGuidance_has_EmsArtefactItem,
        EmsProgramGuidanceID.Id in RelevantProgramGuidanceIDs
    ).EmsArtefactItemID
);






//LookUp(EmsArtefactItem, ID=EmsArtefactItemID.Id, Group)

ClearCollect(CurrentItemIds,
    AddColumns(Filter(
        EmsArtefactMetadata_has_EmsArtefactItem,
        EmsArtefactMetadataID.Value = Text(DropdownArtefact.Selected.ID)
        //EmsArtefactItemID.Value
    ), GROUP, LookUp(EmsArtefactItem, ID in EmsArtefactItemID.Value, Group))
);


If(
        IsBlank(dependentRecord),
        "",
        "Dependency on " & dependentRecord.Title & " Issue " & dependentRecord.Issue
    )
)



Filter(EmsArtefactMetadata, Status.Value = "Active")




{
        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",
        Id: EditMeetingChanges_Left.Selected.ID,
        Value: Text(EditMeetingChanges_Left.Selected.ID)
    }






// Prevent unneccesary updates to data table
If (
    dataHasChanged,
    Set(
        firstRecord,
        First(CprEntries)
    );
    Set(
        lastRecord,
        First(
            Sort(
                CprEntries,
                ID,
                SortOrder.Descending
            )
        )
    );
    Set(
        iterationsNo,
        RoundUp(
            (lastRecord.ID - firstRecord.ID) / 500,
            0
        )
    );
    ClearCollect(iterations, Sequence(iterationsNo,0));
    Clear(AllRecordsNew);
    ForAll(
        iterations,
        With(
            {
                prevThreshold: Value(Value) * 500 + firstRecord.ID - 1,
                nextThreshold: (Value(Value) + 1) * 500 + firstRecord.ID - 1
            },
            If(
                lastRecord.ID > Value,
                Collect(
                    AllRecordsNew,
                    AddColumns(
                        Filter(
                            CprEntries,
                            IdDuplicate > prevThreshold && IdDuplicate <= nextThreshold
                        ),
                        dCOAW_Text,
                        Dcoaw.DisplayName,
                        CertificationEngineer_Text,
                        CertificationEngineer.DisplayName
                    )
                )
            )
        )
    );
    ClearCollect(AllRecords, AllRecordsNew);
    Set(dataHasChanged, false);
);


Download
ClearCollect(CPR_Collection, 
    RenameColumns(
        DropColumns(
            AddColumns(
                Filter(
                    SortByColumns(AllRecords, "ID", If(SortDescending1, SortOrder.Ascending, SortOrder.Descending)),
                    (Program.Value = ProgramFilter.Selected || !ProgramFilter.Toggle),
                    (ActivityType.Value = ActivityFilter.Selected || !ActivityFilter.Toggle),
                    (Classification.Value = ClassificationFilter.Selected || !ClassificationFilter.Toggle),
                    (Status.Value = StatusFilter.Selected || !StatusFilter.Toggle),
                    (CreatedDate >= StartedFilter.FromDate || !StartedFilter.Toggle),
                    (CreatedDate <= StartedFilter.ToDate || !StartedFilter.Toggle),
                    (ClosedDate >= ClosedFilter.FromDate || !ClosedFilter.Toggle),
                    (ClosedDate <= ClosedFilter.ToDate || !ClosedFilter.Toggle),
                    (TextSearchBox.Text = "" ||
                        (
                            Lower(TextSearchBox.Text) in Lower(Title) || 
                            Lower(TextSearchBox.Text) in Lower(Dcoaw.DisplayName) || 
                            Lower(TextSearchBox.Text) in Lower(CertificationEngineer.DisplayName)
                        )
                    )
                ),
                // Columns to Add
                ActivityType_Export, ActivityType.Value,
                Classification_Export, Classification.Value,
                Status_Export, Status.Value,
                Program_Export, Program.Value,
                Plm_Export, If(IsBlank(PlmReference), "Nil.", PlmReference),
                DcoawDisplay, If(IsBlank(Dcoaw.DisplayName), "Not Assigned", Dcoaw.DisplayName),
                CertFocalDisplay, If(IsBlank(CertificationEngineer.DisplayName), "Not Assigned", CertificationEngineer.DisplayName),
                RaisedDateDisplay, If(IsBlank(CreatedDate), "Missing Data!", Text(CreatedDate, "dd/mm/yyyy")),
                ClosedDateDisplay, If(Status.Value = "Closed",
                                        If(IsBlank(ClosedDate), "Missing Data!", Text(ClosedDate, "dd/mm/yyyy")),
                                        If(Status.Value = "Cancelled", "Cancelled", "Open")
                                    ),
                Comments_Export, If(IsBlank(Comments), "Nil.", TrimEnds(Comments))
            ),
        
            //Columns to Drop
            ID, Dcoaw, 'Created By', Created, 'Modified By', Modified, ClassificationReference, CertificationEngineer, ApprovalReference, 'File name with extension', 'Full Path', 'Has attachments', Identifier, IsFolder, 'Link to item', Name, 'Folder path', Thumbnail, 'Version number', EditingBy, EditingTime, CertificationEngineer_Text, dCOAW_Text, IdDuplicate, Program, Classification, Status, ActivityType, ClosedDate, CreatedDate,'{ContentType}', PlmReference, Comments),
        // Columns to Rename
        Cpn, '(01) CPN',
        Plm_Export, '(02) Other Ref (PLM)',
        Title, '(03) Title',
        Program_Export, '(04) Aircraft Type/s',
        ActivityType_Export, '(05) Activity Type',
        Classification_Export, '(06) Classification',
        Status_Export, '(07) Status',
        RaisedDateDisplay, '(08) Raised',
        ClosedDateDisplay, '(09) Closed',
        CertFocalDisplay, '(10) Cert. Focal',
        DcoawDisplay, '(11) dCOAW',
        Comments_Export, '(12) Comments')
    
    );
Set(ExportJSON, JSON(CPR_Collection,JSONFormat.IncludeBinaryData & JSONFormat.IgnoreUnsupportedTypes));
Set(
    varCSVFile,
    'CPRExcelExport'.Run(JSON(CPR_Collection,JSONFormat.IncludeBinaryData & JSONFormat.IgnoreUnsupportedTypes)).linkoutput
);
Download(varCSVFile);


"For Approval: " &
Concat(
    ForAll(
        Filter(
            EccbMeetings,
            ThisItem.ID in ForAll(ThisRecord.ForApproval, ThisRecord.Id)
        ),
        $"{ThisRecord.Board.Value} {ThisRecord.Title}, held {ThisRecord.HeldMeetingDate}"
    ),
    Value,
    Char(13)
) & Char(10) &
"For Endorsement: " &
Concat(
    ForAll(
        Filter(
            EccbMeetings,
            ThisItem.ID in ForAll(ThisRecord.ForEndorsement, ThisRecord.Id)
        ),
        $"{ThisRecord.Board.Value} {ThisRecord.Title}, held {ThisRecord.HeldMeetingDate}"
    ),
    Value,
    Char(13)
) & Char(10) &
"For Review and IA: " &
Concat(
    ForAll(
        Filter(
            EccbMeetings,
            ThisItem.ID in ForAll(ThisRecord.ForReviewAndIa, ThisRecord.Id)
        ),
        $"{ThisRecord.Board.Value} {ThisRecord.Title}, held {ThisRecord.HeldMeetingDate}"
    ),
    Value,
    Char(13)
)



    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForReviewAndIa,
        LookUp(
            EccbChangeRequest,
            Class.Value = "1" && ID = Id
        ).Title
    ),



-  Manual HTML table

"<table style=""width: 100%; border: 1px solid black; border-collapse: collapse"">
    <tr style=""border: 1px solid black; border-collapse: collapse""></tr>
        <th style=""color: white; background-color: #808080; border: 1px solid black; border-collapse: collapse"">Action</th>
        <th style=""color: white; background-color: #808080; border: 1px solid black; border-collapse: collapse"">Requesting approval to proceed to review</th>
        <th style=""color: white; background-color: #808080; border: 1px solid black; border-collapse: collapse"">Requesting approval</th>
        <th style=""color: white; background-color: #808080; border: 1px solid black; border-collapse: collapse"">Presenting for endorsement</th>
    </tr>
    <tr style=""border: 1px solid black; border-collapse: collapse"">
        <td style=""color: white; background-color: #0059b3; border: 1px solid black; border-collapse: collapse"">Class 1</td>
        <td style=""border: 1px solid black; border-collapse: collapse"">
            <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForReviewAndIa,
        LookUp(
            EccbChangeRequest,
            Class.Value = "1" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
        <td style=""border: 1px solid black; border-collapse: collapse"">
                    <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForApproval,
        LookUp(
            EccbChangeRequest,
            Class.Value = "1" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
        <td style=""border: 1px solid black; border-collapse: collapse"">
        <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForEndorsement,
        LookUp(
            EccbChangeRequest,
            Class.Value = "1" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
    </tr>
    <tr style=""border: 1px solid black; border-collapse: collapse"">
        <td style=""color: white; background-color: #049560; border: 1px solid black; border-collapse: collapse"">Class 2</td>
                <td style=""border: 1px solid black; border-collapse: collapse"">
            <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForReviewAndIa,
        LookUp(
            EccbChangeRequest,
            Class.Value = "2" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
        <td style=""border: 1px solid black; border-collapse: collapse"">
                    <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForApproval,
        LookUp(
            EccbChangeRequest,
            Class.Value = "2" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
        <td style=""border: 1px solid black; border-collapse: collapse"">
        <ul>    
    " & Concat(
    ForAll(
        LookUp(
            EccbMeetings,
            ID = MeetingId
        ).ForEndorsement,
        LookUp(
            EccbChangeRequest,
            Class.Value = "2" && ID = Id
        ).Title
    ),
    If(
        IsBlank(Value),
        "",
        "<li>" & Value & "</li><br>"
    )
) & "</ul>
        </td>
    </tr>
</table>"
```

- Mutiple ForAll :  ThisRecord refers to the inner most loop. so the outer loop need to be aliased using AS
Clear(cImpactSummary);
Clear(cIARequired);
ForAll(
    ChangeSummaryQuickView_Gallery.Selected.ProgramApplicability.Value As PA,
    If(
        EndsWith(
            PA.Value,
            "(ALL)"
        ),
        Collect(
            cIARequired,
            Filter(
                cChoicesImpactedBU,
                First(
                    Split(
                        PA.Value,
                        "("
                    )
                ).Value in Value
            )
        ),
        Collect(
            cIARequired,
            PA.Value
        )
    )
);
Clear(cIASubmited);
ForAll(
    ChangeSummary_ImpactGallery_1.AllItems As Impact,
    ForAll(
        Impact.Program,
        Collect(
            cIASubmited,
            {Program: ThisRecord.Value}
        )
    )
);
ForAll(
    cChoicesImpactedBU,
    Collect(
        cImpactSummary,
        {
            Program: ThisRecord.Value,
            IARequired: If(
                ThisRecord.Value in cIARequired.Value,
                "Yes",
                "No"
            ),
            IASubmitted: If(
                ThisRecord.Value in cIASubmited.Program,
                "Yes",
                "No"
            )
        }
    )
);

- Vertical Text

"<div
style='
text-align:left;
position:absolute;
left:"&-Round((Self.Height-Self.Width/2),0)+100&"px;
top: "&Round((Self.Height-Self.Width/2),0)-100&"px;
width: "&Self.Height&"px;
height: "&Self.Width&"px;
transform: rotate(90deg);
border:0.5px;
border-style: solid;
font-weight:" &If(EndsWith(ThisItem.Program, "(All)"), "bold", "normal") &";
'>
&nbsp" & ThisItem.Program &"
</div>"

- Patch Collections 

ClearCollect(
    CurrentItems,
    Switch(
        EditChangeList_Tabs.Selected.Value,
        "For Endorsement",
        CurrentMeeting.ForEndorsement,
        "For Review and IA",
        CurrentMeeting.ForReviewAndIa,
        "For Approval",
        CurrentMeeting.ForApproval.Id
    )
);
Collect(
    CurrentItems,
    {
        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference",
        Id: EditMeetingChanges_Left.Selected.ID,
        Value: Text(EditMeetingChanges_Left.Selected.ID)
    }
);
Set(
    CurrentMeeting,
    Patch(
        CurrentMeeting,
        Switch(
            EditChangeList_Tabs.Selected.Value,
            "For Endorsement",
            {ForEndorsement: CurrentItems},
            "For Review and IA",
            {ForReviewAndIa: CurrentItems},
            "For Approval",
            {ForApproval: CurrentItems}
        )
    )
);

### concatanation

Concat(LookUp(OcePrograms, ThisItem.Program))

### Power automate Filter query and condition check

ID gt 8 and ID lt 11

if(equals(split(item(), ',')?[7], ''), null, split(item(), ',')?[7])


item()?['field_3']
Items(`EccbActionLoop')['field_5']


if(empty(variables('varAssigned')) and not(empty(items('EccbActionsLoop')?['field_3'])), concat('This action was assigned to ', items('EccbActionsLoop')?['field_3']), null)



if(and(empty(variables('varAssigned')), greater(length(items('EccbActionsLoop')?['field_3']), 0)), concat('This action was assigned to ', items('EccbActionsLoop')?['field_3']), null)

### Htmltext:


"<a href=""" & LookUp(EmsArtefactItem, ID=ThisItem.EmsArtefactItemID.Value, Link) & """>" & LookUp(EmsArtefactItem, ID=ThisItem.EmsArtefactItemID.Value, Title) & "</a>"



