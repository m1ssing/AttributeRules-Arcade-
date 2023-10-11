Attribute Rules Documentation
**Horizon.DBO.CenterlineEDIT**

**Calculation**

**Centerline ID - Calculate Centerline ID by increasing the value by 1**
-- NextSequenceValue("cntln_id")

**Split Intersecting Road - Splits intersecting roads at their intersection and the address ranges will be updated to reflect where the split occured. (This Rule is Disabled)**
// This rule will split intersecting roads at their intersection and the address ranges will be updated to reflect where the split occured

// Define the Road Centerline fields
var centerlineid_field = "cntlnid";
var precenterlineid_field = "precenterlineid"
var fromleft_field = "fromleft";
var fromright_field = "fromright";
var toleft_field = "toleft";
var toright_field = "toright";

//Define any fields to be copied from the centerline when split (lower case)
var centerline_field_names = ["rclnguid", "discrpagid", "rangeprefixleft", "fromleft", "toleft", "parityleft", "rangeprefixright", "fromright", "toright", "parityright", "fullname", "fedroute", "fedrtetype", "afedrte", "afedrtetype", "stroute", "strtetype", "astrte", "astrtetype", "ctyroute", "onewaydir", "roadlevel", "inwater", "roadclass", "countryleft", "countryright", "stateleft", "stateright", "countyleft", "countyright","munileft", "muniright", "zipleft", "zipright", "msagleft", "msagright", "esnleft", "esnright"]

// Check if the line feature is used to manual split the intersecting road and not be added to the layer
// Otherwsie don't run the rule if the road was added by this rule from a previous insert
var manualSplit = false;
if (!HasKey($feature, precenterlineid_field)) return;
if ($feature[precenterlineid_field] == "Manual Split") {
    manualSplit = true;
}
else if (!IsEmpty($feature[precenterlineid_field])) 
{
    return;
}

// Get the global id and geometry from the road
var globalid = $feature.globalid;
var geom = Geometry($feature);

var adds = [];
var updates = [];
var deletes = [];

var segments = [];

// This function calculates a new from and to address based on the percentage along the line the split occurs
function newToFrom(from, to, percent) {
    if (from == null || to == null) return [null, null];

    var range = Abs(to - from);
    if (range < 2) return [from, to];

    var val = percent * range;
    var newVal = 0;

    if ((Floor(val) % 2) == 0) newVal = Floor(val);
    else if ((Ceil(val) % 2) == 0) newVal = Ceil(val);
    else newVal = Floor(val) - 1;

    if (newVal == range) newVal -= 2;

    if (from > to) return [from - newVal, from - newVal - 2];
    else return [from + newVal, from + newVal + 2];
}

// This function splits a road using another road and returns an array of 2 geometries
// If a valid split does not occur it returns null
function splitRoad(road, splitRoad) {
    // Cut the intersecting road and return if the result of the cut is less than 2 features
    var newRoads = Cut(road, splitRoad);
    if (Count(newRoads) < 2) return;

    var validCut = true;
    var geometries = []

    // Loop through collection of lines and check that it was a valid cut in the middle of a segment
    for (var i in newRoads) {
        if (newRoads[i] == null || Length(newRoads[i]) == 0) {
            validCut = false;
            continue;
        }

        // Handle multipart geometries
        var allParts = MultiPartToSinglePart(newRoads[i]);
        for (var p in allParts) {
            Push(geometries, allParts[p]);
        }
    }

    // Process the cut if valid
    if (validCut) {

        var firstGeometry = null;
        var secondGeomArray = [];
        var firstPoint = road.paths[0][0];

        // Loop through each geometry in the cut
        // Store the geometry including the first vertex of the orginal road as the first geometry
        // Collect all other geometries in an array
        for (var i in geometries) {
            if (Equals(firstPoint, geometries[i].paths[0][0])) {
                firstGeometry = geometries[i];
            } else {
                Push(secondGeomArray, geometries[i]);
            }
        }

        // Merge all other geometries as the second geometry
        var secondGeometry = Union(secondGeomArray);
        return [firstGeometry, secondGeometry];
    }
    return;
}

// This function breaks the feature at all intersections with other roads in the dataset and populates an array of geometries
function breakRoadAtIntersections(geom, intersectingRoads) {
    // Test if a split occured
    var splitOccured = false;
    for (var i in intersectingRoads) {    
        var geometries = splitRoad(geom, intersectingRoads[i]);
        if (IsEmpty(geometries)) continue;
        
        // If the two geometries are returned from the split process each to see if the can be split again
        splitOccured = true;
        breakRoadAtIntersections(geometries[0], intersectingRoads);
        breakRoadAtIntersections(geometries[1], intersectingRoads);
        break;
    }
    // If no split occured add the geometry to the segments array
    if (!splitOccured) {
        Push(segments, geom);
    }
}

var intersectingRoads = []
for (var road in Intersects(FeatureSetByName($datastore, "Horizon.DBO.CenterlineEDIT"), geom)) {
    if (globalid == road.globalid || Equals(geom, Geometry(road))) continue;
    Push(intersectingRoads, road);
}
if (manualSplit) {
    Push(deletes, {'globalID': globalid})
    Push(segments, geom);
}
else {
    breakRoadAtIntersections(geom, intersectingRoads);
}

for (var i in segments) {
    // Update the geometry of the original feature to be the first segment from the array 
    if (i == 0) {
        geom = segments[i];
    }
    else {
        // Store an add for a new road for each additional segment and copy the attributes from the original feature
        var featureAttributes = Dictionary(Text($feature))['attributes'];
        var newAttributes = {};
        for (var k in featureAttributes) {
            if (IndexOf(centerline_field_names, Lower(k)) > -1 && featureAttributes[k] != null) {
                newAttributes[k] = featureAttributes[k];
            } else {
                continue;
            }
        }
        // Update the precenterlineid field attribute so this rule is not re-run for this new segment
        newAttributes[precenterlineid_field] = "New";
        Push(adds, {
            'attributes': newAttributes,
            'geometry': segments[i]
        })
    }
}

// Split the roads using the new feature segments
for (var r in intersectingRoads) {
    var road = intersectingRoads[r];
    for (var i in segments) {
        var geometries = splitRoad(Geometry(road), segments[i]);
        if (IsEmpty(geometries)) continue;
        
        var firstGeometry = geometries[0];
        var secondGeometry = geometries[1];

        // Get the address range of the intersecting road
        var fromRight = road[fromright_field];
        var toRight = road[toright_field];
        var fromLeft = road[fromleft_field];
        var toLeft = road[toleft_field];

        // Calculate the new address ranges based on the intersection location along the line
        var geometryPercent = Length(firstGeometry, 'feet') / (Length(firstGeometry, 'feet') + Length(secondGeometry, 'feet'));
        var newToFromLeft = newToFrom(fromLeft, toLeft, geometryPercent)
        var newToFromRight = newToFrom(fromRight, toRight, geometryPercent)

        // Store an update for the intersecting road with the first geometry from the cut and the new right to and left to value 
        var attributes = {}
        if (newToFromRight[0] != null) attributes[toright_field] = newToFromRight[0];
        if (newToFromLeft[0] != null) attributes[toleft_field] = newToFromLeft[0];
        Push(updates, {
            'globalID': road.globalid,
            'attributes': attributes,
            'geometry': firstGeometry
        })

        // Store an add for a new road with the second geometry from the cut and the new right from and left from value 
        var featureAttributes = Dictionary(Text(road))['attributes'];
        var newAttributes = {};
        for (var k in featureAttributes) {
            if (Lower(k) == fromright_field && newToFromRight[1] != null) {
                newAttributes[fromright_field] = newToFromRight[1];
            } else if (Lower(k) == fromleft_field && newToFromLeft[1] != null) {
                newAttributes[fromleft_field] = newToFromLeft[1];
            } else if (IndexOf(centerline_field_names, Lower(k)) > -1 && featureAttributes[k] != null) {
                newAttributes[k] = featureAttributes[k];
            } else {
                continue;
            }
        }
        newAttributes[precenterlineid_field] = road[centerlineid_field];
        Push(adds, {
            'attributes': newAttributes,
            'geometry': secondGeometry
        })
        
        break;
    }
}

// Using the edit parameter return the list of updates and adds for the split roads and add alias names
return {
    "result": {
        "geometry": geom                  
    },
    'edit': [
        {'className': 'Horizon.DBO.CenterlineEDIT', 'adds': adds, 'updates': updates, 'deletes': deletes}
    ]
};

**Copy Alias Name - Copies the road alias names from original road centerline after a split.**

// This rule will copy the road alias names from original road centerline after a split.

// Define the Road Centerline fields
var centerlineid_field = "CNTLN_ID";
var precenterlineid_field = "precenterlineid";

// Define the Road Name Aliases fields
var aliascenterlineid_field = "CNTLN_ID";

// Define any fields to be copied from the road name aliases table (lower case)
var alias_field_names = ["PRE_DIR", "PRE_TYPE", "STREET_NAM", "STREET_TYP", "SUF_DIR"]

If (!HasKey($feature, precenterlineid_field)) return;

// If the previous centerline id is blank or null return
var id = $feature[precenterlineid_field];
If (IsEmpty(id)) return;

// Find all the related road alias names for the split road
// Store an add for every road alias and related it to the new road that was added after the split
var adds = [];
var roadNameAliases = Filter(FeatureSetByName($datastore, "Horizon.DBO.Address_AltName", alias_field_names, false), centerlineid_field + " = @id");
for (var roadNameAlias in roadNameAliases) {
    var featureAttributes = Dictionary(Text(roadNameAlias))['attributes'];
    var newAttributes = {};
    for (var k in featureAttributes) {
        if (IndexOf(alias_field_names, Lower(k)) > -1 && featureAttributes[k] != null) {
            newAttributes[k] = featureAttributes[k];
        } else {
            continue;
        }
    }
    newAttributes[aliascenterlineid_field] = $feature[centerlineid_field]
    Push(adds, {
        'attributes': newAttributes
    })
}

// Using the edit parameter return the list of updates and adds for the split roads and add alias names
return {
    'edit': [{'className': 'Horizon.DBO.Address_altName', 'adds': adds}]
};

**Update Site Addresses - When the road name changes find all site addresses that fall within the address range and update their road name (This Rule is Disabled)**

// This rule will run when the road name changes, find all site addresses that fall within the address range and update their road name

// Define Road Centerline fields
var fullname_field = "fullname";
var fromleft_field = "fromleft";
var fromright_field = "fromright";
var toleft_field = "toleft";
var toright_field = "toright";
var munileft_field = "munileft";
var muniright_field = "muniright";

// Define the Site Addresses fields
var addressfullname_field = "fullname";
var addrnum_field = "addrnum";
var municipality_field = "municipality";

// If the full road name is unchanged return;
If (!HasKey($feature, fullname_field)) return;
var fullname = $feature[fullname_field];
var origFullName = $originalFeature[fullname_field]
if (origFullName == fullname) return;

var fromLeft = $feature[fromleft_field];
var toLeft = $feature[toleft_field];
var fromRight = $feature[fromright_field];
var toRight = $feature[toright_field];

// This function will return the parity (0-0, Even, Odd, Both) given the from and to values of a side
function getParity(from, to) {
    if (IsEmpty(from) || from == 0 || IsEmpty(to) || to == 0) {
  if (from == 0 && to == 0) {
   return ["0-0", 0, 0];
  }
  return ["Error", null, null];
 }
    var minval = Min([from, to]);
 var maxval = Max([from, to]);
 
    if (from % 2 == 0 && to % 2 == 0) return ["Even", minval, maxval];
    
    if (from % 2 != 0 && to % 2 != 0) return ["Odd", minval, maxval];
    
    return ["Both", minval, maxval];
}

function updateSiteAddress(updates, siteAddress) {
 Push(updates, {
  'globalID': siteAddress[globalid_field],
  'attributes': Dictionary(addressfullname_field, fullname)
 })
}

var parityLeft = getParity(fromLeft, toLeft);
var parityRight = getParity(fromRight, toRight);

// If the road has no odd or even ranges return
if (Includes(["0-0", "Error"], parityLeft[0]) && Includes(["0-0", "Error"], parityRight[0])) return;

// Find all site addresses that have the same road name as road name prior to the edit
// Add each matching site address to an array storing the global id and updated road name
var updates = []
var siteAddresses = Filter(FeatureSetByName($datastore, "Horizon.DBO.AddressEDIT", [addressfullname_field, addrnum_field, municipality_field], false), addressfullname_field + " = @origFullName");
var globalid_field = Schema(siteAddresses).globalIdField;
for (var siteAddress in siteAddresses) {
    // Test if the address number is a number, if not continue
    var addrnum = Number(siteAddress[addrnum_field])
    if (isNaN(addrnum)) {
        continue;
    }
    
 if (siteAddress[municipality_field] == $feature[munileft_field] && (addrnum >= parityLeft[1] && addrnum <= parityLeft[2])) {
  if (parityLeft[0] == "Both") {
   updateSiteAddress(updates, siteAddress);
  }
  else if (parityLeft[0] == "Odd" && addrnum % 2 != 0) {
   updateSiteAddress(updates, siteAddress);
  }
  else if (parityLeft[0] == "Even" && addrnum % 2 == 0) {
   updateSiteAddress(updates, siteAddress);
  }
 }
 
 if (siteAddress[municipality_field] == $feature[muniright_field] && (addrnum >= parityRight[1] && addrnum <= parityRight[2])) {
  if (parityRight[0] == "Both") {
   updateSiteAddress(updates, siteAddress);
  }
  else if (parityRight[0] == "Odd" && addrnum % 2 != 0) {
   updateSiteAddress(updates, siteAddress);
  }
  else if (parityRight[0] == "Even" && addrnum % 2 == 0) {
   updateSiteAddress(updates, siteAddress);
  }
 }
}

// Using the edit parameter return the list of updates for the site address points
return {
    'edit': [
        {'className': 'Horizon.DBO.AddressEDIT', 'updates': updates}
    ]
};

**Left & Right Parity - Calculates the left and right parity of the road (THis Rule is Disabled)**
//This rule will calculate the left and right parity of the road

// Define the Road Centerline fields
var fromleft_field = "fromleft";
var toleft_field = "toleft";
var fromright_field = "fromright";
var toright_field = "toright";
var parityleft_field = "parityleft";
var parityright_field = "parityright";

// This function will return the parity (0-0, Even, Odd, Both) given the from and to values of a side
function getParity(from, to) {
    if (IsEmpty(from) || from == 0 || IsEmpty(to) || to == 0) {
        if (from == 0 && to == 0) {
            return "Z";
        }
        return null;
    }
    
    if (from % 2 == 0 && to % 2 == 0) return "E";
    
    if (from % 2 != 0 && to % 2 != 0) return "O";
    
    return "B"
}

// If the road from left field is missing return
if (!HasKey($feature, fromleft_field)) return;

var parityleft = getParity($feature[fromleft_field], $feature[toleft_field]);
var parityright = getParity($feature[fromright_field], $feature[toright_field]);

return {
    "result": {
        "attributes":
            Dictionary(
                parityleft_field, parityleft,
                parityright_field, parityright
            )
    }
}

**Update Validation Status - Mark roads or nearby site addresses as requiring validation if a road is deleted or has its road name, address range, municipality or geometry updated (This Rule is Disabled)**
// This rule will mark roads or nearby site addresses as requiring validation if a road is deleted or has its road name, address range, municipality or geometry updated

// Specify default search distance for related site addresses (feet)
var search_distance = 1000;

// Define the Road Centerline fields
var id_field = "cntlnid";
var fullname_field = "fullname";
var fromleft_field = "fromleft";
var toleft_field = "toleft";
var fromright_field = "fromright";
var toright_field = "toright";
var munileft_field = "munileft";
var muniright_field = "muniright";

// This function will return whether the id, road name, address range, municipality, or geometry were updated
function isFeatureUpdated() {
    if (!Equals(Geometry($feature), Geometry($originalFeature))) return true;
    if ($feature[id_field] != $originalFeature[id_field]) return true;
    if ($feature[fullname_field] != $originalFeature[fullname_field]) return true;
    if ($feature[fromleft_field] != $originalFeature[fromleft_field]) return true;
    if ($feature[toleft_field] != $originalFeature[toleft_field]) return true;
    if ($feature[fromright_field] != $originalFeature[fromright_field]) return true;
    if ($feature[toright_field] != $originalFeature[toright_field]) return true; 
    if ($feature[munileft_field] != $originalFeature[munileft_field]) return true;
    if ($feature[muniright_field] != $originalFeature[muniright_field]) return true; 
    return false;
}

// If the edit was an update and the one of the defined properties was not updated, return
if (!HasKey($feature, fullname_field)) return;
if ($editcontext.editType == "UPDATE" && !isFeatureUpdated()) return;

var ids = [$feature[id_field], $originalFeature[id_field]]
var fullname = $feature[fullname_field];
var fullname_orig = $originalFeature[fullname_field];
var munileft = $feature[munileft_field];
var munileft_orig = $originalFeature[munileft_field];
var muniright = $feature[muniright_field];
var muniright_orig = $originalFeature[muniright_field];

var search_string = id_field + " IN @ids" + " OR " + fullname_field + " = @fullname" + " OR " + fullname_field + " = @fullname_orig"
var roadCenterlines = Filter(FeatureSetByName($datastore, "Horizon.DBO.CenterlineEDIT", ["objectid"], false), search_string)
var road_objectIDs = [];
for (var road in roadCenterlines) {
    // Test that the road id matches the feature id prior or after the edit or that it is in the same municipality or intersects the feature
    if (Includes(ids, road[id_field]) || Intersects(road, $feature) || Includes([munileft, munileft_orig], road[munileft_field]) || Includes([muniright, muniright_orig], road[muniright_field])) {
        Push(road_objectIDs, road.objectid);  
    }    
}

// Get site addresses within 1000 feet of the road
var siteAddresses = Intersects(FeatureSetByName($datastore, "Horizon.DBO.AddressEDIT", ["objectid"], false), Buffer($feature, search_distance));
var address_objectIDs = [];
for (var siteAddress in siteAddresses) {
    Push(address_objectIDs, siteAddress.objectid);
}

// Mark the roads and site addresses as requiring validation
return {
    'validationRequired': [{
        'classname': 'Horizon.DBO.centerlineEDIT',
        'objectIDs': road_objectIDs
    },{
        'classname': 'Horizon.DBO.addressEDIT',
        'objectIDs': address_objectIDs
    }]
}

**Constraint**

**Invalid Full Road Name - Full road name is not defined in the Master Road Name table**
// This rule will ensure the full name exist in the Master Road Name table
// It will compare the left and right municipality to the municipality in the Master Road Name table
// If the left and right municipality are different there will need to be a road name for each municipality in the Master Road Name table

// Define the Road Centerline fields
var fullname_field = "fullname";
var munileft_field = "munileft";
var muniright_field = "muniright";

// Define the Master Road Names fields
var masterfullname_field = "fullname";
var mastermuni_field = "municipality";

// If the fullname is blank or null return
If (!HasKey($feature, fullname_field)) return true;
var fullname = $feature[fullname_field];
If (IsEmpty(fullname)) return true;

// Get the left and right side municipalities
var munileft = $feature[munileft_field];
var muniright = $feature[muniright_field];
var municipalities = [munileft, muniright];

// This function will attempt to find partial matches in the master road name table
function findPartialMatches(search_municipalities) {
    //Attempt to find partial matches and return in error message
    var partialMatches = [];
    var fullname_parts = Split(fullname, ' ', -1, true)
    
    for (var i in fullname_parts) {
        if (Count(fullname_parts[i]) < 3) continue;
        
        var search_string = "%" + fullname_parts[i] + "%";
        var masterRoadNames = Filter(FeatureSetByName($datastore, "Horizon.DBO.Address_MasterRoads", [masterfullname_field, mastermuni_field], false), mastermuni_field + " IN @search_municipalities" + " AND " + masterfullname_field + " LIKE @search_string");
        for (var road in masterRoadNames) {
            var roadname = `${road[masterfullname_field]} (${road[mastermuni_field]})`
            if (!Includes(partialMatches, roadname)) {
                Push(partialMatches, roadname );
            }    
        }
    }
    
    return partialMatches;
}

// Search the master road name table for a row matching the fullname and municipality
var masterRoadNames = Filter(FeatureSetByName($datastore, "Horizon.DBO.Address_MasterRoads", [masterfullname_field, mastermuni_field], false), mastermuni_field + " IN @municipalities" + " AND " + masterfullname_field + " = @fullname");
// If the left and right side municipality we only need one matching record
// If no matching records are found return an error
if (munileft == muniright) {
    if (Count(masterRoadNames) == 0) {   
        //Attempt to find partial matches and return in error message
        var partialMatches = findPartialMatches(municipalities);
        if (Count(partialMatches) == 0) return {"errorMessage" : "Match for left and right municipality not found. No partial matches found." };
        return {"errorMessage" : "Match for left and right municipality not found. Partial matches: " + Concatenate(partialMatches, ", ")};
    }
}
// If left and right side municipality are different, we need one record for each municipality in the table
else {
    var leftmatch = null;
    var rightmatch = null;
    for (var road in masterRoadNames) {
        if (road[mastermuni_field] == munileft) leftmatch = `${road[masterfullname_field]} (${road[mastermuni_field]})`;
        if (road[mastermuni_field] == muniright) rightmatch = `${road[masterfullname_field]} (${road[mastermuni_field]})`;
    }
    
    // If either the left or the right side municipality is not found return an error
    if (IsEmpty(leftmatch) || IsEmpty(rightmatch)) {
        var error = "Match for left and right municipality not found. "
        var search_municipalities = municipalities;
        if (IsEmpty(leftmatch) && !IsEmpty(rightmatch)) {
            error = "Match for left municipality not found. ";
            search_municipalities = [munileft];
        }
        if (!IsEmpty(leftmatch) && IsEmpty(rightmatch)) {
            error = "Match for right municipality not found. ";
            search_municipalities = [muniright];
        }
        
        var partialMatches = findPartialMatches(search_municipalities);
        if (Count(partialMatches) == 0) return {"errorMessage" : error + "No partial matches found." };
        return {"errorMessage" : error + "Partial matches: " + Concatenate(partialMatches, ", ")};
    }
}
return true;

**Invalid Geometry - Road has a null or multipart geometry**

// This rule will ensure the road doesn't have a null or multipart geometry

var geom = Geometry($feature);
if (IsEmpty(geom)) {
    return false;
}
if (Count(geom["paths"]) > 1) {
    return false;
}
return true;

**Dupe Centerline ID - Check for duplicateCenterline ID**

// This rule will validate that the road ID is unique

// Define the Road Centerline fields
var id_field = "cntlnid";

// If the id_field is blank or null return
If (!HasKey($feature, id_field)) return true;
var id = $feature[id_field];
If (IsEmpty(id)) return true;

// Search the site addresses feature class for records with the same full address.
// If there is more than one return an error
var roads = Filter(FeatureSetByName($datastore, "Horizon.DBO.CenterlineEDIT", [id_field], false), id_field + " = @id");
if (Count(roads) > 1) {
    return {"errorMessage" : `(${id})`}
}
return true;

**Centerline Master Roads - Check if Centerline Name is in Master Roads table**
var masterRoads = FeatureSetByName($datastore,
    "Horizon.DBO.Address_MasterRoads", ["fullname"], false);
    var nameArray = [];
    for (var road in masterRoads){
    var masterName = road.fullname
    Push(nameArray, Upper(masterName))
    }
    var check = Includes(nameArray, Upper($feature.fullname))
    IF (check==false){
    return false;
    }
    return true;
