---
name: add-migration
description: Generate a Sequelize migration file with proper timestamp, transaction wrapping, and up/down methods.
---

Generate a properly formatted Sequelize migration file.

## Step 1: Gather requirements

Ask the user for the intent. Common operations:

1. **Create table** — entity name, columns with types
2. **Add column** — table name, column name, type, nullable, default
3. **Remove column** — table name, column name
4. **Rename column** — table name, old name, new name
5. **Add index** — table name, column(s), unique?
6. **Add foreign key** — table name, column, references table
7. **Insert seed data** — table name, rows
8. **Modify column** — table name, column name, new type/constraints

## Step 2: Generate timestamp

```bash
date +%Y%m%d%H%M%S
```

## Step 3: Generate migration file

Create `server/migrations/{timestamp}-{description}.js`:

### Create table example:

```javascript
"use strict";

/** @type {import('sequelize-cli').Migration} */
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.createTable(
                "{EntityModels}",
                {
                    id: {
                        type: Sequelize.BIGINT,
                        allowNull: false,
                        autoIncrement: true,
                        primaryKey: true,
                    },
                    // ... columns
                },
                { transaction: t }
            );
        });
    },

    async down(queryInterface) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.dropTable("{EntityModels}", { transaction: t });
        });
    },
};
```

### Add column example:

```javascript
"use strict";

/** @type {import('sequelize-cli').Migration} */
module.exports = {
    async up(queryInterface, Sequelize) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.addColumn(
                "{TableName}",
                "{columnName}",
                {
                    type: Sequelize.STRING(255),
                    allowNull: true,
                    defaultValue: null,
                },
                { transaction: t }
            );
        });
    },

    async down(queryInterface) {
        await queryInterface.sequelize.transaction(async (t) => {
            await queryInterface.removeColumn("{TableName}", "{columnName}", {
                transaction: t,
            });
        });
    },
};
```

## Step 4: Verify naming

- Filename: `{YYYYMMDDHHmmss}-{kebab-case-description}.js`
- Table names match Sequelize model class names (e.g., `StationModels`, `UserRolesModels`)
- Description is concise: `create-stations`, `add-stationId-to-users`, `add-vehicle-permission-area`

## Rules

- **Always CommonJS** (`module.exports`), not ESM
- **Always wrap in transaction** (`queryInterface.sequelize.transaction`)
- **Always include both `up` and `down`** — `down` must reverse `up` exactly
- **Type conventions**:
  - IDs: `Sequelize.BIGINT`
  - Strings: `Sequelize.STRING(N)` with explicit length
  - Booleans: `Sequelize.BOOLEAN`
  - Timestamps: `Sequelize.DATE`
  - JSON: `Sequelize.JSON`
- **Foreign keys**: Use `references: { model: "TableName", key: "id" }` and `onDelete`/`onUpdate`
- **Seed data**: Use `queryInterface.bulkInsert` inside the transaction
- Pattern: see existing files in `server/migrations/`
