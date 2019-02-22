(This document is a WIP)

# Pre-requisites

- [Docker](https://www.docker.com/products/docker-desktop) - Run containers
- [VSCode](https://code.visualstudio.com/) - Code editor

# Commands

1. Open a command line terminal

2. Create a new folder `gql-example` on your computer and `cd` into it

3. Let's run a `bash` terminal using a docker `ruby:2.4` image _(~1 minute on a good connection)_

   - Mac / Windows PowerShell
   ```bash
   docker run -p 3000:3000 --rm --name rubydocker -v $(PWD):/app/gql-example -w /app/gql-example -it ruby:2.4 bash
     ```

   - Windows (cmd)
   ```DOS
   docker run -p 3000:3000 --rm --name rubydocker -v %cd%:/app/gql-example -w /app/gql-example -it ruby:2.4 bash
   ```

   This text will refer to this terminal window as _rubydocker terminal_ from now on.

4. Using _rubydocker terminal_, install nodejs (required by graphql generator) & rails `5.1` _(~1 minute)_
   ```rails
   apt-get update
   apt-get install nodejs -y
   gem install rails -v 5.1
   ```

   _(Note: there are more elegant ways to create a docker image with rails and node pre-installed, out of the scope of this tutorial)_

5.  Using _rubydocker terminal_, initialize a lightweight version of a rails application _(~2 minute)_
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

# Part 2

11. Open _VSCode_ and open your `gql-example` local directory

12. Using _VSCode_, open `./app/controllers/graphql_controller.rb` and change the following lines:

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

13. Using _VSCode_, create a new file `./app/graphql/static_field_resolver.rb` with these lines:

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
          obj[method_name]
        else
          raise "No field resolver for #{type.name}\##{field.name}"
        end
      end
    end
    ```
    _(Note: there are more optimized ways to achieve similar behavior, out of the scope of this tutorial)_


14. Using _VSCode_, create a new file `./schema.graphql` with these lines:

    ```
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
    ```

15. Using _VSCode_, create a new file `./app/graphql/types/group_type.rb` with these lines:

    ```
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


16. Using _VSCode_, replace the entire content of `./app/graphql/types/query_type.rb` with these lines:

    ```
    module Types
      class QueryType < Types::BaseObject
        def group(id:)
          {id: id, display_name: "Banana#{id}"}
        end
      end
    end
    ```
