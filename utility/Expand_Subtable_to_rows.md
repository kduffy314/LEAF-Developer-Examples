# Expand Subtables (Grid input widget) into Rows

Reports that include Subtables don't export seamlessly into Excel. Instead of using Excel's PowerQuery to parse JSON data, this will expand subtable items into new rows.

In the example below, the report of all Property Passes contains a subtable with each individual piece of equipment. By expanding each equipment into its own row, it's easier to handle in Excel.

```html
<script>
let CSRFToken = '<!--{$CSRFToken}-->';

function scrubHTML(input) {
    if (input == null) {
        return '';
    }
    let t = new DOMParser().parseFromString(input, 'text/html').body;
    return t.textContent;
}

function updateTitle(title) {
    if (title != '') {
        let siteName = document.querySelector('#headerDescription')?.innerText;
        let siteLocation = document.querySelector('#headerLabel')?.innerText;
        if (siteName == undefined) {
            document.querySelector('title').innerText = scrubHTML(`${title}`);
        }
        else {
            document.querySelector('title').innerText = scrubHTML(`${title} - ${siteName} | ${siteLocation}`);
        }
    }
}

async function getData() {
    // want records that have expired or will expire in 1 month
    let newDate = new Date();
    newDate.setMonth(newDate.getMonth() + 1);
    let expiredDate = `${newDate.getUTCMonth() + 1}/${newDate.getUTCDate()}/${newDate.getUTCFullYear()}`;

    // Create a new Query
    let query = new LeafFormQuery();

    query.addTerm('categoryID', '=', 'form_aaf61');
    query.addTerm('stepID', '!=', 'resolved');
    query.addDataTerm('data', '36', '<=', expiredDate);
    query.addTerm('deleted', '=', 0);
    query.join('status');
    query.join('stepFulfillment');
    query.getData([38, 36, 32]);

    query.onProgress(numProcessed => {
        document.querySelector('#grid').innerHTML = `Processing ${numProcessed}+ records`;
    });

    // Execute the query
    let result = await query.execute();

    // expand contents of sub-table into new rows
    let parsed = {};
    let counter = 1;
    for (let i in result) {
        for (let j in result[i].s1.id38_gridInput.cells) {
            if (parsed[counter] == undefined) {
                //parsed[counter] = result[i];
                parsed[counter] = {};
            }

            // formGrid expects unique recordIDs
            parsed[counter].realRecordID = i;
            parsed[counter].recordID = counter;
            parsed[counter].title = result[i].title;
            parsed[counter].stepFulfillment = result[i].stepFulfillment;
            parsed[counter].s1 = result[i].s1;

            parsed[counter].deviceEE = result[i].s1.id38_gridInput.cells[j][0];
            parsed[counter].serial = result[i].s1.id38_gridInput.cells[j][1];
            parsed[counter].type = result[i].s1.id38_gridInput.cells[j][2];
            parsed[counter].manufacturer = result[i].s1.id38_gridInput.cells[j][3];
            parsed[counter].model = result[i].s1.id38_gridInput.cells[j][4];
            parsed[counter].value = result[i].s1.id38_gridInput.cells[j][5];

            counter++;
        }
    }

    // Initialize the Grid
    let formGrid = new LeafFormGrid('grid'); // 'grid' maps to the associated HTML element ID

    // This enables the Export button
    formGrid.enableToolbar();
    formGrid.hideIndex();

    // Required to initialize data for the grid
    formGrid.setData(Object.keys(parsed).map(key => parsed[key]));
    formGrid.setDataBlob(parsed);

    // The column headers are configured here
    formGrid.setHeaders([
        /*{name: 'Title', indicatorID: 'title', callback: function(data, blob) { // The Title field is a bit unique, and must be implemnted this way
            $('#'+data.cellContainerID).html(blob[data.recordID].title);
        }},*/
        {name: 'UID', indicatorID: 'uid', callback: (data, blob) => {
                let realRecordID = blob[data.recordID].realRecordID;
                $('#' + data.cellContainerID).html(`<a href="index.php?a=printview&recordID=${realRecordID}">${realRecordID}</a>`);
        }},
        {name: 'User Email', indicatorID: 32},
        //{name: 'Property on Temporary Loan', indicatorID: 38}, // this would normally show up as a sub-table
        {name: 'Device EE', indicatorID: 'deviceEE', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].deviceEE);
        }},
        {name: 'Serial / IMEI', indicatorID: 'serial', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].serial);
        }},
        {name: 'Type', indicatorID: 'type', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].type);
        }},
        {name: 'Manufacturer', indicatorID: 'manufacturer', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].manufacturer);
        }},
        {name: 'Model', indicatorID: 'model', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].model);
        }},
        {name: 'Value', indicatorID: 'value', editable: false, callback: (data, blob) => {
                $('#' + data.cellContainerID).html(blob[data.recordID].value);
        }},
        {name: 'Date Signed', indicatorID: 'step15', editable: false, callback: function (stepID) {
                return function (data, blob) {
                    if (blob[data.recordID].stepFulfillment != undefined
                        && blob[data.recordID].stepFulfillment[stepID] != undefined) {
                        var date = new Date(blob[data.recordID].stepFulfillment[stepID].time * 1000);
                        $('#' + data.cellContainerID).html(date.toLocaleDateString());
                    }
                }
            }(15)
        },
        {name: 'Expected Return Date', indicatorID: 36},
    ]);

    // Load and populate the spreadsheet
    formGrid.sort('recordID', 'desc');
    formGrid.renderBody();
    formGrid.announceResults();
}

async function main() {

    getData();

    updateTitle('Overdue Property Passes');
}

// Ensures the webpage has fully loaded before starting the program.
document.addEventListener('DOMContentLoaded', main);
</script>

<h1>Overdue Property Passes and Loaned Property</h1>
<p>Includes passes that will expire within 1 month</p>
<div id="grid">Loading...</div>
```
