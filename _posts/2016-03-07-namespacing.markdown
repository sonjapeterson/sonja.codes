---
layout: post
title:  "Namespacing Your Rails Models Without Losing Your Mind"
date:   2016-03-07 23:42:49 -0500
tags: ["Ruby", "Ruby on Rails"]
---

Recently, I was working on a new Rails app for work that I felt had a good potential use case for namespacing. A made up (not perfect) example of the type of situation I was in would be modeling a bunch of different games, which each involved some models that might have similar names--like a ChessMove and PokerMove--but different attributes, relationships and behavior, making them totally separate classes.

In this situation, instead of having a bunch of classes like ChessMove, ChessPiece, ChessPieceType, etc and getting super sick of typing "Chess" over and over again, you could use namespacing and have a Chess module with Move, Piece and PieceType classes in it. In order to avoid conflicts with a Move class for a different game like Poker, you'd want to keep the prefix for your database table. But inside your namespaced classes, you can just do `piece.move` instead of all this `chess_move.chess_piece` nonsense. Namespacing would also give a nice, clear structure to your app/models folder, with the models relevant to a particular domain clearly grouped together.

Namespacing models in this way isn't a completely typical Rails thing to do, so it does require some additional configuration, especially for associations between models. Also, other gems that interact closely with your models might not play well with it. It still felt worth it for the particular requirements of my app, but you should definitely take a minute here to make sure that it's worth it for yours.

Once I decided to use namespacing, here's the changes I had to make, with some made-up examples using chess\*:

- Create a folder with the name of the namespace and move all of the models to it.
- Add the module to each model's class definition:

``` ruby
  module Chess
    class Move < ActiveRecord::Base
      [...]
    end
  end
```

- Define the appropriate table names for each class--ActiveRecord doesn't consider the module path when creating the table name:

``` ruby
module Chess
  class Move < ActiveRecord::Base
    self.table_name = "chess_moves"
  end
end
```

- Update the namespaced model's associations to other namespaced models to remove the prefix and explicitly specify the class_names and foreign keys. You'll have to specify the foreign key involved for BOTH has_many and belongs_to associations.

``` ruby
  module Chess
    class Move < ActiveRecord::Base
      belongs_to :piece, class_name: Chess::Piece, foreign_key: "chess_piece_id"
    end
  end
```

- Update your class names everywhere you use them to the namespaced version (Chess::Piece instead of ChessPiece), and update the associations to the new, prefix-less names you've defined.

- Update your ActiveRecord scopes. This is a little confusing. I always forget that `includes` and `references` use the name you've defined for the association (so no namespace prefix), but where clauses involving associations want the real table names. So a query like this before namespacing:

``` ruby
  class ChessMove < ActiveRecord::Base
    scope :used_pawn, -> { includes(:chess_piece).where(chess_piece: { type: "pawn" }) }
  end
```

  becomes this:

``` ruby
  module Chess
    class Move < ActiveRecord::Base
      scope :used_pawn, -> { includes(:pieces).where(chess_piece: { type: "pawn" }) }
    end
  end
```

- If you're using FactoryGirl in your specs, you'll need to set the class option:

``` ruby
FactoryGirl.define do
  factory :chess_move, class: Chess::Move do
    [...]
  end
end
```

For namespaced associations in your factories, you'll need to specify the factory explicitly:

``` ruby
factory :chess_move do
  association :piece, factory: :chess_piece
end
```

Doing each of these steps for all 7 of my namespaced models seemed fairly verbose and tedious, so I created a NamespacedModel concern to do them automatically. It's pretty ugly, but it does the job:

``` ruby
module NamespacedModel
  extend ActiveSupport::Concern

  included do
    def self.table_name
      self.name.delete("::").underscore.pluralize
    end

    # creates an association for a model in the same namespace
    def self.namespaced_association(assoc_type, assoc_name, other_opts={})
      assoc_class = assoc_name.to_s.classify
      opts = {class_name: "#{self.module_prefix}::#{assoc_class}"}.merge(other_opts)
      if assoc_type == :belongs_to
        associated_table_name = "#{self.module_prefix.delete("::").underscore}_#{assoc_name}"
        opts[:foreign_key] ||= "#{associated_table_name}_id"
      elsif [:has_one, :has_many].include? assoc_type
        opts[:foreign_key] ||= "#{table_name.singularize}_id"
      end
      self.send(assoc_type, assoc_name, opts)
    end

    def self.module_prefix
      self.name.deconstantize
    end
  end

end
```

Then in your models you can just do

``` ruby
module Chess
  class Move
    include NamespacedModel
    namespaced_association(:belongs_to, :piece)
  end
end
```

You could also have handled the table name situation by making a file like this, called `app/models/chess.rb`:

``` ruby
module Chess
  def self.table_name_prefix
    "chess_"
  end
end
```

But if you have more than one namespace to deal with and you want to also set up namespaced associations, using a concern to do it seems like a more generic and easy solution.

That's (pretty much) it! Make sure all your tests pass, then admire how much nicer your code and models directory looks.

![#flawless](http://i.giphy.com/zBxaKpZA1tsli.gif)

\*Disclaimer: this examples might show a bad way to model a chess game, since that wasn't the actual domain I was dealing with.