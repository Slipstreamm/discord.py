.. _discord_ui_overview:

.. currentmodule:: discord

``discord.ui`` -- User Interface Components
===========================================

Discord's UI kit provides ways to build interactive interfaces using components like buttons, select menus, and modals. The
:mod:`discord.ui` module in this library offers high level abstractions to create these interfaces easily.

This page presents a basic overview of how to build views, lay out components, and respond to user interaction.

Creating a View
---------------

A :class:`~discord.ui.View` groups components together and manages the interaction
lifecycle. Views can be persistent or temporary and support custom timeouts.

.. code-block:: python3

    from discord.ext import commands
    import discord

    bot = commands.Bot(command_prefix="!")

    class Confirm(discord.ui.View):
        def __init__(self):
            super().__init__(timeout=30)
            self.value = None

        @discord.ui.button(label="Confirm", style=discord.ButtonStyle.green)
        async def confirm(self, interaction: discord.Interaction, button: discord.ui.Button):
            self.value = True
            self.stop()

        @discord.ui.button(label="Cancel", style=discord.ButtonStyle.red)
        async def cancel(self, interaction: discord.Interaction, button: discord.ui.Button):
            self.value = False
            self.stop()

    @bot.command()
    async def ask(ctx: commands.Context):
        view = Confirm()
        await ctx.send("Do you wish to continue?", view=view)
        await view.wait()
        if view.value is None:
            await ctx.send("Timed out...")
        elif view.value:
            await ctx.send("Confirmed!")
        else:
            await ctx.send("Cancelled.")

Handling Select Menus
---------------------

Select menus allow members to choose from a list of options. They are represented by
:class:`~discord.ui.Select` and can be defined inside a view with the
:func:`~discord.ui.select` decorator.

.. code-block:: python3

    class FruitView(discord.ui.View):
        @discord.ui.select(
            placeholder="Choose your favourite fruit",
            min_values=1,
            max_values=1,
            options=[
                discord.SelectOption(label="Apple", value="apple"),
                discord.SelectOption(label="Banana", value="banana"),
                discord.SelectOption(label="Orange", value="orange"),
            ],
        )
        async def select_callback(self, interaction: discord.Interaction, select: discord.ui.Select):
            await interaction.response.send_message(f"You chose {select.values[0]}!")

Buttons and Modals
------------------

Buttons are created with :class:`~discord.ui.Button` and can be declared using the
:func:`~discord.ui.button` decorator. For more complex input, :class:`~discord.ui.Modal`
provides a dialog style interface with text inputs.

.. code-block:: python3

    class Feedback(discord.ui.Modal, title="Feedback"):
        name = discord.ui.TextInput(label="Name")
        message = discord.ui.TextInput(label="Feedback", style=discord.TextStyle.paragraph)

        async def on_submit(self, interaction: discord.Interaction):
            await interaction.response.send_message("Thanks for your feedback!", ephemeral=True)

    class FeedbackView(discord.ui.View):
        @discord.ui.button(label="Give Feedback", style=discord.ButtonStyle.primary)
        async def feedback(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.send_modal(Feedback())

Combining Everything
--------------------

Views can contain multiple components arranged in rows. Each view keeps track of its
children and automatically disables components when timed out.

More information about the available classes and their APIs can be found in the
:doc:`interactions/api` reference under :ref:`discord_ui_kit`.

Persistent Views
----------------

Sometimes a view should remain active even after the bot restarts. To achieve
this, register the view with :meth:`discord.Client.add_view` and ensure all
components provide a ``custom_id``. Persistent views cannot have a timeout set.

.. code-block:: python3

    class Poll(discord.ui.View):
        def __init__(self):
            super().__init__(timeout=None)

        @discord.ui.button(label="Vote", style=discord.ButtonStyle.primary,
                           custom_id="poll:vote")
        async def vote(self, interaction: discord.Interaction,
                       button: discord.ui.Button):
            await interaction.response.send_message(
                "Thanks for voting!", ephemeral=True
            )

    bot.add_view(Poll())

Timeouts and Disabling Components
---------------------------------

When a view times out you can disable the components to prevent further
interaction. Override :meth:`~discord.ui.View.on_timeout` and edit the message
that contains the view to update it.

.. code-block:: python3

    class AutoDisable(discord.ui.View):
        async def on_timeout(self) -> None:
            for item in self.children:
                item.disabled = True

            await self.message.edit(view=self)

Component Layout and Row Management
-----------------------------------

Each message can contain up to five rows of components with five components in
each row. By default, items are appended in the order they are defined but you
can explicitly control the layout using the ``row`` keyword argument or by
manually adding items with :meth:`~discord.ui.View.add_item`.

.. code-block:: python3

    class RowExample(discord.ui.View):
        def __init__(self):
            super().__init__()
            self.add_item(discord.ui.Button(label="Top", row=0))
            self.add_item(discord.ui.Button(label="Bottom", row=4))

        @discord.ui.button(label="Middle", row=2)
        async def middle(self, interaction: discord.Interaction, _: discord.ui.Button):
            await interaction.response.send_message("Middle row button")

Handling Interaction Responses
------------------------------

Interactions need a response within three seconds. If your callback requires
long processing you should defer the response to buy yourself more time.

.. code-block:: python3

    class DeferExample(discord.ui.View):
        @discord.ui.button(label="Process", style=discord.ButtonStyle.blurple)
        async def process(self, interaction: discord.Interaction, button: discord.ui.Button):
            await interaction.response.defer(thinking=True)
            await lengthy_operation()
            await interaction.followup.send("Done!", ephemeral=True)

Pre-defined Select Menus
------------------------

In addition to :class:`~discord.ui.Select`, there are convenience subclasses for
common types of selections such as channels, roles, users, and mentionables.
These can be constructed manually or by passing the ``cls`` keyword parameter to
the :func:`~discord.ui.select` decorator.

.. code-block:: python3

    class SelectView(discord.ui.View):
        @discord.ui.select(cls=discord.ui.ChannelSelect, channel_types=[discord.ChannelType.voice])
        async def channel(self, interaction: discord.Interaction, select: discord.ui.ChannelSelect):
            await interaction.response.send_message(f"Chosen channel: {select.values[0]}")

Pagination Example
------------------

The following example demonstrates how the building blocks come together to
create a paginated message. Clicking the buttons will edit the message in-place
to show the next or previous page.

.. code-block:: python3

    class Paginator(discord.ui.View):
        def __init__(self, pages: list[str]):
            super().__init__(timeout=60)
            self.pages = pages
            self.index = 0

        async def update_message(self, interaction: discord.Interaction):
            content = self.pages[self.index]
            await interaction.response.edit_message(content=content, view=self)

        @discord.ui.button(label="Prev", style=discord.ButtonStyle.secondary)
        async def previous(self, interaction: discord.Interaction, _button: discord.ui.Button):
            self.index = (self.index - 1) % len(self.pages)
            await self.update_message(interaction)

        @discord.ui.button(label="Next", style=discord.ButtonStyle.secondary)
        async def next(self, interaction: discord.Interaction, _button: discord.ui.Button):
            self.index = (self.index + 1) % len(self.pages)
            await self.update_message(interaction)

    @bot.command()
    async def paginate(ctx: commands.Context):
        view = Paginator(["Page 1", "Page 2", "Page 3"])
        await ctx.send("Page 1", view=view)

See Also
--------

More examples can be found in the :resource:`examples <examples>` directory.
Refer to the :doc:`interactions/api` page for an exhaustive API reference.
