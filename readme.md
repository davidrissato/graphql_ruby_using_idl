(This document is a draft - WIP)

# Objectives
This guide will show how to replace the default [class based API](https://graphql-ruby.org/schema/class_based_api) schema declaration by adding a [GraphQL SDL](https://graphql.org/learn/schema/#type-language) schema declaration support<sup>[*1](#f1)</sup> to a [Ruby GraphQL](https://graphql-ruby.org) rails based application.

I could have only created a generator (and I might do that later on), but I believe documenting this approach might make it easier to understand what is going on behind the scenes.

<a id="f1"><sup>*1</sup></a>: SDL means Schema Definition Language, and it is also sometimes referred as GraphQL IDL - Interface Definition Language.

# Pre-requisites

- [Docker](https://www.docker.com/products/docker-desktop) - Run containers
- [VSCode](https://code.visualstudio.com/) - Code editor

# Part 1 - Creating a base rails install and a minimal web application

## Creating a Rails docker image pointing to a local folder

1. Open a command line terminal

2. Create a new folder `gql-example` on your computer and `cd` into it

3. Let's run a `bash` terminal using a docker `ruby:2.4` image _(~1 minute on a good connection)_

   _Note: This step is creating a non-persistent docker image pointing to a local folder, just to guarantee that we can use the exactly same commands on all environments (Windows / Mac / Linux). In case you are already running a Debian based `bash` terminal, and have Ruby 2.4 installed (I strongly recommend using [RVM](https://rvm.io/) for that), feel free to skip this step._

   - Mac / Windows PowerShell
   ```bash
   docker run -p 3000:3000 --rm --name rubydocker -v $(PWD):/app/gql-example -w /app/gql-example -it ruby:2.4 bash
     ```

   - Windows (cmd)
   ```DOS
   docker run -p 3000:3000 --rm --name rubydocker -v %cd%:/app/gql-example -w /app/gql-example -it ruby:2.4 bash
   ```

   This text will refer to this terminal window as _rubydocker terminal_ from now on.

4. Using _rubydocker terminal_, install nodejs (required by graphql generator to install graphiql) & rails `5.1` _(~1 minute)_
   ```rails
   apt-get update
   apt-get install nodejs -y
   gem install rails -v 5.1
   ```

   _(PS: there are more elegant ways to create a docker image with rails and node pre-installed, out of the scope of this tutorial)_

## Create a minimal GraphQL rails application

5.  Using _rubydocker terminal_, initialize a lightweight rails application _(~2 minute)_
    ```bash
    rails new . --skip-active-record --skip-yarn --skip-action-mailer --skip-active-storage --skip-action-cable --skip-coffee --skip-javascript --skip-turbolinks --skip-bootsnap --skip-spring
    ```

6. Using _rubydocker terminal_, add graphql gem dependency and use [GraphQL rails generator](https://graphql-ruby.org/schema/generators.html#graphqlinstall) to create the basic structure for GraphQL:
   ```bash
   bundle add graphql
   bundle install
   rails generate graphql:install
   bundle install
   ```
## Running a smoke test

7. Using _rubydocker terminal_, run rails application

   ```bash
   rails s
   ```

8. Open your browser in your local machine and visit (http://localhost:3000/graphiql). It should open the _graphiql interface_.

9. Using _graphiql interface_, execute the following query:
    ```graphql
    query {
      testField
    }
    ```
10. Using _rubydocker terminal_, stop application using `<CTRL> + C`
----

# Part 2 - Replacing schema declaration

_This section is an intermediary and simplified step that I created to show what exacttly is being replaced. If you are only interested on adding support without too many details on what is being replaced, feel free to jump directly to part 3_

11. Open _VSCode_ and open your `gql-example` local directory

12. Using _VSCode_, create a new file `./schema.graphql` with these lines:

    ```graphql
    schema {
      query: Query
    }

    type Query {
      testField: String!
    }
    ```
    
    **Explanation:** _This is a schema definition file using GraphQL IDL that matches the exact declaration that has been created by GraphQL rails generator for field `testField` from type Query_

13. Using _VSCode_, replace the entire content of `./app/graphql/types/query_type.rb` with these lines:

    ```diff
      module Types
        class QueryType < Types::BaseObject
    -     # Add root-level fields here.
    -     # They will be entry points for queries on your schema.
    -
    -     # TODO: remove me
    -     field :test_field, String, null: false,
    -       description: "An example field added by the generator"
          def test_field
            "Hello World!"
          end
        end
      end
    ```
    
    **Explanation:** _We are removing field declaration from this class because we don't need it anymore. Notice that we kept the field resolver intact._


14. Using _VSCode_, open `./app/controllers/graphql_controller.rb` and change the following lines:

    ```diff
    context = {
      # Query context goes here, for example:
      # current_user: current_user,
    }
    - result = GqlExampleSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
    + result = schema.execute(query, variables: variables, context: context, operation_name: operation_name)
    render json: result
    ```

    ```diff
    private
    +
    + def schema
    +   @@schema ||= GraphQL::Schema.from_definition(IO.read('./schema.graphql'), default_resolve: StaticFieldResolver)
    + end
    ```
    
    **Explanation:** _We are replacing the original schema definition by a file based version, and also assigning a `default_resolver` (this class will be the created in the next step)._

15. Using _VSCode_, create a new file `./app/graphql/static_field_resolver.rb` with these lines:

    ```ruby
    class StaticFieldResolver
      def self.call(type, field, obj, args, ctx)
        type_class = Object.const_get("Types::#{type.name}Type")
        method_name = field.name.underscore
        instance = type_class.authorized_new(obj, ctx)
        instance.public_send(method_name)
      end
    end
    ```

    **Explanation:** _This class is used to map types and fields to their field resolvers. I'm following the same conventions defined by the generator, where types are declared on a package named 'Types::' and the class name is composed by the type name, followed by the word 'Type'. Ex: 'Types::QueryType'. This is a very simple implementation that will be optimized in the next part of this guide_

16. Remove these two files from `/app/graphql/`: `gql_example_schema.rb` and `mutation_type.rb`. (We don't need them anymore)

17. Run a new smoke test by executing again [Running a smoke test](#running-a-smoke-test) (Part 1 - Steps 7 to 10)
----

# Part 3 - Adding support to arguments on the schema based resolution

18. (Execute this step only if you have skiped part 2): Remove these two files from `/app/graphql/`: `gql_example_schema.rb` and `mutation_type.rb`. (We don't need them anymore)

19. Open _VSCode_ and open your `gql-example` local directory

20. Using _VSCode_, create/replace file `./schema.graphql` with these lines:

    ```graphql
    schema {
      query: Query
    }

    type Query {
      group(id: String): Group
    }

    type Group {
      id: String
      displayName: String
    }
    
    type Dog {
      id: String
      name: String
      breed: String
    }
    ```

21. Using _VSCode_, replace the entire content of `./app/graphql/types/query_type.rb` with these lines:

    ```ruby
    module Types
      class QueryType < Types::BaseObject
        def group(id:)
          {id: id, display_name: "Banana#{id}"}
        end
      end
    end
    ```

22. Using _VSCode_, create a new file `./app/graphql/types/group_type.rb` with these lines:

    ```ruby
    module Types
      class GroupType < Types::BaseObject
        def id  
          object[:id]
        end

        def display_name
          object[:display_name]
        end
      end
    end
    ```

23. Using _VSCode_, open `./app/controllers/graphql_controller.rb` and change the following lines (if you haven't done it already in part 2):

    ```diff
    context = {
      # Query context goes here, for example:
      # current_user: current_user,
    }
    - result = GqlExampleSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
    + result = schema.execute(query, variables: variables, context: context, operation_name: operation_name)
    render json: result
    ```

    ```diff
    private
    +
    + def schema
    +   @@schema ||= GraphQL::Schema.from_definition(IO.read('./schema.graphql'), default_resolve: StaticFieldResolver)
    + end
    ```

24. Using _VSCode_, create/replace file `./app/graphql/static_field_resolver.rb` with these lines:

    ```ruby
    class StaticFieldResolver
      def self.call(type, field, obj, args, ctx)
        type_class = Object.const_get("Types::#{type.name}Type")
        method_name = field.name.underscore
        instance = type_class.authorized_new(obj, ctx)
        if instance.respond_to?(method_name)
          args = field.arguments.map { |k,v| [k.to_sym, args[k]] }.to_h
          if args.empty?
            instance.public_send(method_name)
          else
            instance.public_send(method_name, **args)
          end
        elsif obj.is_a?(Hash)
          obj[method_name.to_sym] || obj[method_name]
        else
          raise "No field resolver for #{type.name}\##{field.name}"
        end
      end
    end
    ```
    _(Note: there are more optimized ways to achieve similar behavior, out of the scope of this tutorial)_
