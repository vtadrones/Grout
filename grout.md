# Grout

## Database Specifications

Grout assumes SQLite version 3.0.0 and higher.

##Design

### Overview

Grout is a scheme for storing tiled map data targeted for mobile devices (Android and iOS) and desktops. It is intended for direct usage and file transfer.  The tiled map data is stored as a tileset in a database. The database may contain multiple tilesets.  Each tileset is identified by a row in the `tilesets` table. A tileset is comprised of multiple tile levels, which are identified by rows in the tileset's `tilelevel_table`. A separate table is used to store the tile information for each tile level. The tiles themselves can be stored directly in the database as blobs or stored in a file container external to the database. One advantage of storing the tiles in an external file container is that it allows large tilesets to be stored on devices with relatively small file size limitations, i.e. 4GB on FAT32 media. An explanation of how to identify a blob or external tileset is described in the **Table Structure** section under **Tile Level Table**.

Grout does not define a method for interacting with the data rendered in the tiles.  If functionality is needed, 

### Schema

#### Tilesets Table

The `tilesets` table contains a listing of all of the Grout tilesets stored in this database.  This allows multiple tilesets to be stored in a single SQLite database.

##### Table Structure

* `tilelevel_table`: (Primary Key) The name of the table in the database that describes all of the level information for the tiles in this tileset.
* `title`: A short informative name for the tileset that might be displayed in a table of contents.
* `description`: A description of the data represented in the tileset.
* `version`: The integer version of the tileset.
* `srid`: The spatial reference (SRS/CRS) of the tileset.
* `format`: The format of the image data, supported options are `jpg` or `png`.
* `bounds`: The maximum extent of the tileset.  The bounds are encoded in WGS84 using the OpenLayer Bounds format: left, bottom, right top.
* **optional** `metadata`: XML document containing geospatial metadata in accordance with ISO 19115 or other metadata standard.

```sql
CREATE TABLE "tilesets" ("tilelevel_table" text PRIMARY KEY,
	"title" text NOT NULL,
	"description" text NOT NULL,
	"version" integer NOT NULL,
	"srid" text NOT NULL,
	"format" text NOT NULL,
	"bbox" text NOT NULL, 
	"metadata" text);
```

#### Tile Level Table

The tile level table stores the information for all zoom levels in a tileset.  This allows for tailoring the scale, resolution, or container of zoom levels.

##### Table Structure

* `tile_table`: (Primary Key) The name of the table in the database that stores the values used to locate the tiles.
* `zoom_level`: The zoom level of this tile level entry.
* `tile_container`: The path to the container used to store the tiles for this tile level.  If the tiles are stored in the tile table as a blob then this value is NULL.
* `pixels_per_side`: The width/height of the tile in pixels.  Tiles must be square.
* `tile_span`: The width/height of a tile in SRS/CRS units.  Tiles must be square.

```sql
CREATE TABLE "sample_tl" ("tile_table" TEXT PRIMARY KEY NOT NULL , 
	"zoom_level" INTEGER NOT NULL ,
	"tile_container" TEXT, 
	"pixels_per_side" INTEGER NOT NULL , 
	"tile_span" DOUBLE NOT NULL );
```

#### Tile Table

The tile table stores the information used to locate the tile.  If the `tile_container` field is NULL in the tile level table then the actual tile data is stored as a blob in this table; **N.B.** this may limit the number of tilesets and/or size of your tileset you can store on some platforms.

##### Table Structure

* `tile_column`: Column index of the tile.
* `tile_row`: Row index of the tile
* **optional** `tile_locator`: Information needed to access tile when stored in an external tile container.  If omitted table must contain `tile_data` field.
* **optional** `tile_data`: Raw tile image data.  If omitted table must contain `tile_locator` field.

```sql
CREATE TABLE "sample_lvl0" ("tile_column" INTEGER NOT NULL , 
	"tile_row" INTEGER NOT NULL , 
	"tile_locator" TEXT NOT NULL , 
	PRIMARY KEY ("tile_column", "tile_row"));
```
**or**
```sql
CREATE TABLE "sample_lvl1" ("tile_column" INTEGER NOT NULL , 
	"tile_row" INTEGER NOT NULL , 
	"tile_data" BLOB NOT NULL , 
	PRIMARY KEY ("tile_column", "tile_row"));
```

### Conversions Examples

#### Converting a Bounding Box to Tile Indices

```javascript
minTileRow = floor(bbox.LowerLeft.X / tile_span);
minTileColumn = floor(bbox.LowerLeft.Y / tile_span);
maxTileRow = floor(bbox.UpperRight.X / tile_span);
maxTileColumn = floor(bbox.UpperRight.Y / tile_span);
```

where `tile_span` is specified in the Tile Level Table for the current zoom level. 

#### Converting Tile Indices to a Bounding Box

```javascript
bbox.LowerLeft.X = minTileRow + tile_span;
bbox.LowerLeft.Y = minTileColumn + tile_span;
bbox.UpperRight.X = maxTileRow + tile_span;
bbox.UpperRight.Y = maxTileColumn + tile_span;
```

where `tile_span` is specified in the Tile Level Table for the current zoom level. 
