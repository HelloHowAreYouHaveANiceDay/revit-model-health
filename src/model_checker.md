---
title: Revit Model Checker
toc: false
---

# Revit Model Checker Demo



## Use Your Model
```js
const file = view(Inputs.file());
```

Export a .csv from your model using [revit-csv](https://github.com/franpossetto/revit-csv), then attach above.

### ---
### ${!file ? 'Showing Sample File': 'Showing User File'}
### ---


```js
var csv_file
if (!file){
    csv_file = await FileAttachment('./usaec_model_checker.csv').csv({typed: true})
} else {
    csv_file = await file.csv({typed: true})
}
```


## Revit Version

<div>${csv_file[0]['RevitVersion']}</div>

## Project Info

```js
const project_in = _.find(csv_file, (r) => r['Category'] == 'Project Information')
```

<div>
    <div>
    </div>
    <div>
        <ul>
            <li><strong>Project Issue Date:</strong> ${project_in['Project Issue Date'] ? new Date(project_in['Project Issue Date']).toISOString().split('T')[0] : ''}</li>
            <li><strong>Organization Name:</strong> ${project_in['Organization Name']}</li>
            <li><strong>Organization Description:</strong> ${project_in['Organization Description']}</li>
            <li><strong>Author:</strong> ${project_in['Author']}</li>
            <li><strong>Client Name:</strong> ${project_in['Client Name']}</li>
            <li><strong>Project Address:</strong> ${project_in['Project Address']}</li>
            <li><strong>Project Name:</strong> ${project_in['Project Name']}</li>
            <li><strong>Project Number:</strong> ${project_in['Project Number']}</li>
        </ul>
    </div>
</div>


## Room Area by Level
```js
// Total area by level in m² and sqft
const roomsByLevel = _.chain(csv_file)
    .filter(r => r.Category === 'Rooms')
    .groupBy('Level')
    .map((rooms, level) => {
        const count = rooms.length;
        const totalArea = _.sumBy(rooms, r => parseFloat(r.Area.split(' ')[0]));
        return {
            Level: level,
            rooms: count,
            area_m2: Math.round((totalArea / 10.7639) * 100) / 100,
            area_sqft: Math.round(totalArea * 100) / 100
        };
    })
    .value();
```

```js
Inputs.table(roomsByLevel, {
    columns: ["Level", "rooms", "area_m2", "area_sqft"],
    header: {
        Level: "Level",
        rooms: "Room Count",
        area_m2: "Area (m²)",
        area_sqft: "Area (sqft)"
    }
})
```


## Health

```js
// Count model groups
const model_groups_count = _.filter(csv_file, (r) => r['Category'] === 'Model Groups').length;

// Count warnings
const views_count = _.filter(csv_file, (r) => r['Category'] === 'Views').length;

// Count elements with empty or null family
const viewport_count = _.filter(csv_file, (r) => r['Category'] === 'Viewports').length;

// Count elements with empty or null type
const sheet_count = _.filter(csv_file, (r) => r['Category'] === 'Sheets').length;
```


<div class="grid grid-cols-4">
  <div class="card"><h1>${model_groups_count}</h1>Model Groups</div>
  <div class="card"><h1>${views_count}</h1>Views</div>
  <div class="card"><h1>${viewport_count}</h1>Viewports</div>
  <div class="card"><h1>${sheet_count}</h1>Sheets</div>
</div>


## Furniture Count

```js
// Count furniture by family and type
const furnitureCount = _.chain(csv_file)
    .filter(r => r.Category === 'Furniture')
    .groupBy('Family and Type')
    .map((items, familyType) => ({
        'Family and Type': familyType,
        'Name': items[0].Name || '',
        'instance_count': items.length
    }))
    .sortBy(item => item['Family and Type'])
    .reverse()
    .value();
```

```js
// Create bar chart for furniture count
const furnitureBarChart = Plot.plot({
    marginLeft: 200,
    height: 400,
    x: {
        label: "Count",
        grid: true
    },
    y: {
        label: null
    },
    marks: [
        Plot.barX(furnitureCount.slice(0, 15), {
            x: "instance_count",
            y: d => d['Name'] || d['Family and Type'],
            sort: {y: "-x"},
            text: d => d.instance_count,
            textAnchor: "end",
        }),
        Plot.text(furnitureCount.slice(0, 15), {
            x: d => d.instance_count,
            y: d => d['Name'] || d['Family and Type'],
            text: d => d.instance_count,
            dx: 5,
            textAnchor: "start"
        }),
        Plot.ruleX([0])
    ]
})
```

```js
furnitureBarChart
```

```js
Inputs.table(furnitureCount, {
    columns: ["Family and Type", "Name", "instance_count"],
    header: {
        "Family and Type": "Family and Type",
        "Name": "Name",
        "instance_count": "Count"
    }
})
```

## Door Families Missing Width

```js
// Doors with missing width
const doorsMissingWidth = _.chain(csv_file)
    .filter(r => r.Category === 'Doors')
    .groupBy(door => {
        return [
            door.width === null, 
            door.Name, 
            door.Family, 
            door.width, 
            door.height
        ].join('||');
    })
    .map((doors, key) => {
        const parts = key.split('||');
        return {
            missing_width: parts[0] === 'true',
            Name: parts[1],
            Family: parts[2],
            width: parts[3] === 'null' ? null : parts[3],
            height: parts[4] === 'null' ? null : parts[4],
            instance_count: doors.length
        };
    })
    .orderBy(['missing_width'], ['desc'])
    .value();
```

```js
Inputs.table(doorsMissingWidth)
```