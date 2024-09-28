---
title: "Creating a Discord Bot - Part 2: Integrating Azure Resources"
date: 2024-09-28T08:00:00-07:00
publish-date: 2024-09-29T12:00:00-07:00
draft: true
categories: ["development"]
tags: ["ts", "js", "docker", "azure", "github", "CI/CD"]
---

This is part 2 in my series of posts that revisits creating a Discord bot. In this post we'll be setting up some resources in Azure and using them in a couple of commands.

You can see all of the posts in this series here:

{{% links/discord-bot %}}

## The Plan

For this part we want to create two more commands in the bot. One for configuring the bot in a server and one for actually creating the events. Since we want this bot to work on multiple servers, we need a way for the server admin to tell the bot which channel to make the event threads under. To do that, we're going to use an Azure Table Store to store the per-server settings.

### Prerequisites

From this point forward, you'll need to have an Azure subscription that we can create the groups in. I'm also using powershell core so any command line commands will be using powershell syntax for things like variables.

## Create The Azure Resources

For this step, I'm going to use the Azure CLI instead of the portal because I prefer to work in a CLI whenever possible. You can find the installation instructions [here](https://learn.microsoft.com/en-us/cli/azure/). You'll also need a Resource Group in the subscription, but I'll show how to create one if you don't have one yet.

### Logging In to Azure

Lets get our terminal open and run this command:

```powershell
az login
```

This will prompt you to log in. Once you've logged in to azure, you can run `az account show` to verify where you've logged into. The output would look something like this:

```json
{
  "environmentName": "AzureCloud",
  "homeTenantId": "{Tenant Id}",
  "id": "{Subscription Id}",
  "isDefault": true,
  "managedByTenants": [],
  "name": "{Subscription Name}",
  "state": "Enabled",
  "tenantId": "{Tenant Id}",
  "user": {
    "name": "{Your email}",
    "type": "user"
  }
}
```

Ensure that the `id` property is your subscription id. If it's not you can run this command:

```powershell
az account set --subscription $SubscriptionId
```

### Create the Resource Group

Before creating the resource group, you'll want to set the name and azure location for it. I'm going to name the group `test-discord-bot` use `westus2` but you can use pretty much any azure region.

If you don't know which regions you can use, you can run this command:

```powershell
az account list-locations -o table
```

Just grab the name from the `Name` column and not the `DisplayName` column.

Let's set the group name to a variable then create the group:

```powershell
$ResourceGroupName = 'test-discord-bot'
az group create --name $ResourceGroupName --location westus2
```

I'm using a variable for the group name because we're going to need it later.

### Create the Storage Account

If you're using the template repo I created, you'll notice a `provisioning` folder, you should `cd` into that directory. If not, you'll want to save the [provisioning bicep file](https://github.com/iamdavidfrancis/discord-bot-template/blob/main/provisioning/provision-storage-account.bicep) somewhere to create the resource.

Back in the terminal, let's set a couple more variables. The `$StorageAccountName` must be unique across all of Azure, so you'll want to choose something unique:

```powershell
$StorageAccountName = 'discordbotstore'
$StorageTableName = 'discordbotdb'
```

Now that those are set, you can trigger a new Azure Deployment by using this command:

```powershell
az deployment group create --resource-group $ResourceGroupName --template-file "./provision-storage-account.bicep" --parameters storageAccountName=$StorageAccountName tableName=$StorageTableName
```

This will create the Storage Account and Table with some default settings I like to use. One of the settings disallows using shared access keys to auth against the storage account, so we'll need to set up a way to access the account when running locally.

To do this, you'll need to give yourself the correct role assignment using Azure RBAC. We'll need your user id, which we can get from the cli using `az ad signed-in-user show`. You can add this to a variable with:

```powershell
$UserId = az ad signed-in-user show --query "id" -o tsv
```

Next, we'll need to note down the role we need: `"Storage Table Data Contributor"`.

```powershell
$RoleName = "Storage Table Data Contributor"
```

Now that we have the user id and role definition id, all we need now is to set the access on the resource. Let's grab the id of the storage account and then assign the role:

```powershell
$StorageAccountId = az storage account show --resource-group $ResourceGroupName --name $StorageAccountName --query "id" -o tsv
az role assignment create --assignee $UserId --role $RoleName --scope $StorageAccountId
```

If everything worked, you should get a response object of the new role assignment. This will allow your user account to access the table data. This means that when we run the bot locally, it can use `DefaultAzureCredential()` to talk to the storage table using your user account from the az cli. This means you should make sure to do an `az login` when opening the terminal to ensure your account is still logged in.

## Use The Resources in the Bot

Now that the storage account is created, let's update the `.env` file with the account and table names you used above.

```env
STORAGE_ACCOUNT_NAME=discordbotstore
STORAGE_TABLE_NAME=discordbotdb
```

Let's make a new folder under `src` called `services` and create two new files:

- `db-schema.ts`
- `db-service.ts`

We'll use `db-schema.ts` to hold the type definitions we're going to use. You can set the content to:

```ts
export interface GuildSettings {
  eventChannelId?: string;
}
```

`db-service.ts` is where we're going to set up the bot to talk to the table.

Let's start by adding some imports:

```ts
import { TableClient, TableEntity } from "@azure/data-tables";
import { DefaultAzureCredential } from "@azure/identity";
import { GuildSettings } from "./db-schema";
import { config } from "../config";
```

The package.json already includes `@azure/data-tables` and `@azure/identity` so you shouldn't have to worry about installing them again. Next up lets add some boilerplate:

```ts
const PartitionKey = "DEFAULT_PARTITION";

class TableDBService {
  private tableClient: TableClient;

  constructor() {}

  public async getGuildSettings(
    guildId: string
  ): Promise<TableEntity<GuildSettings>> {}

  public async addOrUpdateGuildSettings(
    guildId: string,
    settings: TableEntity<GuildSettings>
  ): Promise<void> {}
}

const instance = new TableDBService();

export default instance;
```

Let me explain what's going on here.

All the rows in a table will required a `PartitionKey` and `RowKey`. Since this isn't planned to be a massive db, I'm not worried about partitioning right now, so everything will have the same partition key. If we had a larger table, we would get a performance benefit from calculating some partition key to group servers together. Because of this, I've hard coded a partition id.

I've created a class called `TableDBService` and I'm exporting an instance of it. This means whenever any other part of the app uses this db, it'll be a single instance shared across all of them.

Right now the class can't do anything so let's start adding the implementations.

First off, the constructor. In here we need to create the table client for the other methods to use. We're going to use a `DefaultAzureCredential` to authenticate as it will support the Azure CLI auth we use locally and the managed identity we'll use when we deploy to Azure.

Update the constructor to have:

```ts
constructor() {
  const credential = new DefaultAzureCredential();
  this.tableClient = new TableClient(`https://${config.STORAGE_ACCOUNT_NAME}.table.core.windows.net`, config.STORAGE_TABLE_NAME, credential);
}
```

Now let's implement `getGuildSettings`. This method will fetch the server info from the table.

```ts
public async getGuildSettings(guildId: string): Promise<TableEntity<GuildSettings>> {
  try {
    const result = await this.tableClient.getEntity<TableEntity<GuildSettings>>(PartitionKey, guildId);

    return result;
  }
  catch (error: any) {
    // A 404 is an expected error, it means the row doesn't exist yet.
    // Any other error can be logged for investigation.
    if (error.statusCode !== 404) {
      console.error(error);
    }
  }

  // Include the partition and row keys in the response to make the add/update path easier.
  return {
    partitionKey: PartitionKey,
    rowKey: guildId,
  };
}
```

Because of the way we implemented the get method, `addOrUpdateGuildSettings` is really easy:

```ts
public async addOrUpdateGuildSettings(settings: TableEntity<GuildSettings>): Promise<void> {
  await this.tableClient.upsertEntity(settings, "Merge");
}
```

Now we've got our table service added and ready to go. Let's add some new commands to use them

## Create the New Commands

We want to add two new commands:

1. `/config` to allow server admins to set the channel.
2. `/event` to allow server members to create events.

### Config Command

Let's create a new file named `config.ts` in the `commands` directory. Add these imports at the top:

```ts
import {
  ChatInputCommandInteraction,
  CommandInteraction,
  InteractionContextType,
  PermissionFlagsBits,
  SlashCommandBuilder,
} from "discord.js";
import DBService from "../services/db-service";
```

Once again we'll need to export `data` and `execute`. We will start with `data`. We want the `/config` command to have a subcommand called `set-channel`. This allows us to add more config options later without needing a bunch of top level commands.

```ts
export const data = new SlashCommandBuilder()
  .setName("config")
  .setDescription("Update the bot config")
  .addSubcommand((subcommand) =>
    subcommand
      .setName("set-channel")
      .setDescription("The channel the bot should create event threads in.")
      .addChannelOption((option) =>
        option
          .setName("channel")
          .setDescription("The corresponding channel.")
          .setRequired(true)
      )
  )
  .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild)
  .setContexts(InteractionContextType.Guild);
```

This syntax is very similar to the `/help` command with two exceptions:

1. The `addSubcommand` api. This adds a subcommand named `set-channel` with a description and an option. The option is a `ChannelOption` which tells discord that this option should have a channel picker shown.
2. The `setDefaultMemberPermissions` api. This tells discord that we only want users with "Manage Server" permissions to be able to run the command. Server owners can always override this setting to whatever they want on their server, but this serves as a safe default.

Now let's set up the `execute` function:

```ts
export async function execute(interaction: CommandInteraction) {
  const guildId = interaction.guildId;

  if (!guildId) {
    console.error("The guildId was missing in the interaction.");
    return interaction.reply("Unable to process command.");
  }

  if (interaction.isChatInputCommand()) {
    switch (interaction.options.getSubcommand()) {
      case "set-channel":
        await setChannelHandler(guildId, interaction);
        break;
      default:
        return interaction.reply(
          `Unknown command: ${interaction.options.getSubcommand()}`
        );
    }
  }
}
```

To prevent the method from getting too long, I've set it up so each subcommand has a separate handler method. The main handler just ensures the `guildId` is present and then invokes the subCommand handler. Let's add the `setChannelHandler` implementation:

```ts
async function setChannelHandler(
  guildId: string,
  interaction: ChatInputCommandInteraction
) {
  const guildSettings = await DBService.getGuildSettings(guildId);
  const channel = interaction.options.getChannel("channel");

  if (!channel || !guildSettings) {
    console.error("Channel or guildSettings was missing.");
    return interaction.reply("Unable to process command.");
  }

  guildSettings.eventChannelId = channel.id;

  await DBService.addOrUpdateGuildSettings(guildSettings);
  return interaction.reply(
    `Event Threads will now be posted under ${channel.toString()}`
  );
}
```

In this, we grab the channel id from the interaction, verify that it's been set, then set the property in the DB. We then reply with a success message. One last thing we need to do to enable this command is to add it in the `index.ts` file in the `commands` directory:

```ts
// Existing imports...
import * as config from "./config";

export const commands = {
  // Existing commands...
  config,
};
```

Now let's move on to the `/event/` command.

### Event Command

This is the main function of our bot. The way we want this to work is:

1. User runs `/event` with some options attached.
2. We form an embed with the options from the interaction.
3. We create a thread in the channel we have in the DB.
4. Post the embed in the thread.
5. Reply to the user with a success message.
