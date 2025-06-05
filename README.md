# Gcompris_documentation

# Comparator Activity – Dataset Architecture Documentation

This documentation explains how the **Comparator activity** in GCompris loads and processes its datasets from multiple sources — including fixed, custom, and server-managed datasets. It also describes how datasets are structured, how they are chosen and aggregated, and how the activity starts based on this data.

---

## 📁 Dataset Sources and Their Paths

| Dataset Type      | Storage Location                                                              | Naming Convention                                           | Purpose/Notes                                |
|-------------------|-------------------------------------------------------------------------------|-------------------------------------------------------------|-----------------------------------------------|
| **Fixed**         | `qrc:/gcompris/src/activities/comparator/resource/`                          | Folders named `1`, `2`,...                                 | Predefined levels that ship with the app     |
| **Custom/User**   | `~/.local/share/kde/gcompris-qt/comparator/` (standard qt path)              | Folder named as dataset name in the server `add`           | Datasets added by teachers or users          |
| **Server-db**     | Stored inside SQLite: `~/.local/share/gcompris-teachers/gcompris.sqlite`     | Managed through the GCompris-Teachers UI                   | Server-managed datasets                      |

---

## ⚙️ Dataset Loading Flow

### 1. Activity Initialization

In `ActivityBase.qml`, the activity connects to start and stop signals:

```qml
Component.onCompleted: {
    activity.start.connect(start)
    activity.stop.connect(stop)
}
```

### 2. Level Folder Detection

When datasets (level folders change) are detected, they are sorted and processed:

```javascript
onLevelFolderChanged: {
    if (levelFolder === undefined || levelFolder.length === 0) return;
    datasets = [];
    var data = [];
    // Sort folders numerically
    levelFolder.sort((a, b) => parseInt(a) - parseInt(b));
    // Load datasets from each folder
    for (var level in levelFolder) {
        var id = levelFolder[level];
        var dataset = activityInfo.getDataset(id);
        if (dataset)
            data = data.concat(dataset.data);
    }
    datasets = data;
}
```

### 3. Activity Start

In `Comparator.qml`, the activity starts by calling:

```javascript
onStart: {
    Activity.start(items)
}
```

Which links to the following logic in `comparator.js`:

```javascript
function start(items_) {
    items = items_;
    numberOfLevel = items.levels.length;
    items.currentLevel = Core.getInitialLevel(numberOfLevel);
    initLevel();
}
```

---

## 🧩 Dataset Types and Structures

### 🧷 Fixed Dataset Structure

```json
{
  "shuffle": true,
  "subLevels": [
    { "leftNumber": "13", "rightNumber": "-15" },
    { "leftNumber": "-17", "rightNumber": "16" },
    { "leftNumber": "25", "rightNumber": "8" }
  ]
}
```

### 🎲 Random Dataset Structure

```json
{
  "random": true,
  "minValue": -20,
  "maxValue": 20,
  "numberOfEquations": 10,
  "precision": 1,
  "maxDistanceBetweenNumbers": 5,
  "sameInteger": false,
  "numberOfSublevels": 3
}
```

---

## 🧠 Dataset Selection UI (DialogChooseLevel.qml)

This is the UI where users (or teachers) can choose which datasets are active.

### Initialization Logic:

```javascript
function initialize() {
    chosenLevels = currentActivity.currentLevels.slice();
    difficultiesModel = [];
    for (var level in currentActivity.levels) {
        var data = currentActivity.getDataset(currentActivity.levels[level]);
        difficultiesModel.push({
            level: currentActivity.levels[level],
            enabled: data.enabled,
            objective: data.objective,
            difficulty: data.difficulty,
            selectedInConfig: chosenLevels.includes(currentActivity.levels[level])
        });
    }
}
```

### Selection/Deselection Logic:

Inside `DialogChooseLevel.qml`:

```javascript
onClicked: {
    if (checked) {
        chosenLevels.push(modelData.level);
    } else if (chosenLevels.length > 1) {
        chosenLevels.splice(chosenLevels.indexOf(modelData.level), 1);
    } else {
        checked = true; // prevent deselecting the last active level
    }
}
```

### Saving New Configuration:

```javascript
onSaveData: {
    levelFolder = dialogActivityConfig.chosenLevels;
    currentActivity.currentLevels = dialogActivityConfig.chosenLevels;
    ApplicationSettings.setCurrentLevels(currentActivity.name, dialogActivityConfig.chosenLevels);
    background.start(); // restart activity with updated dataset selection
}
```

---

## 🔁 Full Data Flow Summary

1. **App Launches**: Activity registers start/stop callbacks.

2. **Level Folder Detection**: GCompris scans for dataset folders (both fixed and user).

3. **Dataset Aggregation**: Merged into a single datasets array via `getDataset()`. → in the `activityBase.qml` → inside `onLevelFolderChanges`:

4. **Start Triggered**: `Activity.start()` is called with level info.

5. **Dataset Used**:
   - If fixed: Uses `subLevels`.
   - If random: Generates pairs dynamically.

6. **Models Populated**: QML UI binds to `dataListModel`.

7. **DialogChooseLevel**:
   - User can configure active datasets.
   - Saved in `ApplicationSettings`.
   - Triggers `background.start()` to reload with new config.
