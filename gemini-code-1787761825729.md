# Gramática Libre de Contexto - BattleScript (Formato BNF)

La siguiente gramática describe formalmente la estructura sintáctica del lenguaje BattleScript. Se utiliza la notación BNF (Backus-Naur Form), donde los elementos entre `< >` representan símbolos no terminales, los elementos en mayúsculas representan expresiones regulares (tokens léxicos), y los elementos entre comillas `""` representan terminales literales.

## 1. Símbolos Iniciales y Estructura General
<inicio> ::= <lista_instrucciones>

<lista_instrucciones> ::= <lista_instrucciones> <instruccion> 
                        | <instruccion>

<instruccion> ::= <estrategia> 
                | <partida> 
                | <bloque_main>

## 2. Bloque de Estrategias
<estrategia> ::= <tipo_jugador> ID "{" <accion_inicial> <bloque_reglas> "}"

<tipo_jugador> ::= "mage" 
                 | "warrior"

<accion_inicial> ::= "initial" ":" <accion>

<bloque_reglas> ::= "rules" ":" "[" <lista_reglas> <regla_else> "]"

<lista_reglas> ::= <lista_reglas> <regla> 
                 | <regla>

<regla> ::= "if" <condicion> "then" <accion> ","

<regla_else> ::= "else" <accion>

<accion> ::= "ARCANE_BOLT" | "FIREBALL" | "MAGIC_BARRIER" | "HEALING_RUNE" | "MEDITATE"
           | "SLASH" | "HEAVY_STRIKE" | "SHIELD_BLOCK" | "WAR_CRY" | "REST"

## 3. Expresiones y Condiciones
<condicion> ::= <condicion> "or" <condicion>
              | <condicion> "and" <condicion>
              | "not" <condicion>
              | "(" <condicion> ")"
              | <operando> "==" <operando>
              | <operando> "!=" <operando>
              | <operando> "<" <operando>
              | <operando> ">" <operando>
              | <operando> "<=" <operando>
              | <operando> ">=" <operando>

<operando> ::= INTEGER 
             | FLOAT 
             | "round_number" 
             | "total_rounds" 
             | "self_health" 
             | "opponent_health"
             | "self_resource" 
             | "opponent_resource"
             | "self_score" 
             | "opponent_score"
             | "random"
             | <funcion_historial>
             | <accion>
             | "[" <lista_acciones_secuencia> "]"

<funcion_historial> ::= "get_move" "(" <historial> "," INTEGER ")"
                      | "last_move" "(" <historial> ")"
                      | "get_moves_count" "(" <historial> "," <accion> ")"
                      | "get_moves_count" "(" <funcion_historial> "," <accion> ")"
                      | "get_last_n_moves" "(" <historial> "," INTEGER ")"

<historial> ::= "self_history" 
              | "opponent_history"

<lista_acciones_secuencia> ::= <lista_acciones_secuencia> "," <accion>
                             | <accion>

## 4. Bloque de Partidas (Match)
<partida> ::= "match" ID "{" 
                "players" ":" "[" ID "," ID "]"
                "rounds" ":" INTEGER
                "scoring" ":" "{" <parametros_scoring> "}"
                "bonuses" ":" "{" <parametros_bonuses> "}"
              "}"

<parametros_scoring> ::= "damage_point" ":" INTEGER ","
                         "healing_point" ":" INTEGER ","
                         "successful_defense" ":" INTEGER ","
                         "victory_bonus" ":" INTEGER ","
                         "failed_action_penalty" ":" INTEGER

<parametros_bonuses> ::= "mage_combo" ":" "[" <lista_acciones_secuencia> "]" ","
                         "mage_combo_points" ":" INTEGER ","
                         "warrior_combo" ":" "[" <lista_acciones_secuencia> "]" ","
                         "warrior_combo_points" ":" INTEGER ","
                         "low_health_victory" ":" INTEGER

## 5. Bloque de Ejecución (Main)
<bloque_main> ::= "main" "{" <lista_run> "}"

<lista_run> ::= <lista_run> <instruccion_run>
              | <instruccion_run>

<instruccion_run> ::= "run" "[" <lista_id_partidas> "]" "with" "{" "seed" ":" INTEGER "}"

<lista_id_partidas> ::= <lista_id_partidas> "," ID
                      | ID