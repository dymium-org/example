MATSim integration
================

# Introduction

This example, we show how MATSim, an agent-based traffic simulator in
Java, can be integrated with dymiumCore. We basically send our
individual agents for a ride\!

On a more serious note, this is a use case where we would like to
simulate the impact that our future population would have on the urban
traffic condition in the future years.

This example is a very simple one where we replace an activity-based
model that is usually used for assigning daily travel activities with a
travel demand fusion model. The travel demand fusion model uses the
records in
[VISTA](https://transport.vic.gov.au/about/data-and-research/vista), the
household travel survey of Victoria, and fuses with the individual data
from the simulation.

## Download the `matsim` module

``` r
library(dymiumCore)
download_module("matsim")
```

## Set active scenario

``` r
dymiumCore::set_active_scenario(name = "matsim-integration")
```

A friendly warning, If you feel that the log messages are too much for
you can change it using the follow command.

``` r
library(lgr)
lgr::lgr$set_threshold(level = "warn")
```

To set it back to info level simply do `lgr::lgr$set_threshold(level =
"info")`. But by default the log threshold is set to the ‘info’ level.

## Import events

``` r
library(modules)
library(here)
# Demography
event_demography_age <- modules::use(here::here("modules/demography/age.R"))
event_demography_birth <- modules::use(here::here("modules/demography/birth.R"))
event_demography_death <- modules::use(here::here("modules/demography/death.R"))
event_demography_marriage <- modules::use(here::here("modules/demography/marriage.R"))
event_demography_separation <- modules::use(here::here("modules/demography/separation.R"))
event_demography_divorce <- modules::use(here::here("modules/demography/divorce.R"))
event_demography_breakup <- modules::use(here::here("modules/demography/breakup.R"))
event_demography_cohabit <- modules::use(here::here("modules/demography/cohabit.R"))
event_demography_migration <- modules::use(here::here("modules/demography/migration.R"))
# MATSim
event_matsim_createVISTADemand <- modules::use(here::here('modules/matsim/createVISTADemand.R'))
event_matsim_runcontroler <- modules::use(here::here('modules/matsim/runControler.R'))
```

## Setup a world

``` r
world <- World$new()

# add agents
world$add(x = Population$new(ind_data = toy_individuals, 
                             hh_data = toy_households, 
                             pid_col = "pid", 
                             hid_col = "hid"))

# add models
# demography module's models 
models <- list(fertility = list(yes = 0.05, no = 0.95),
              birth_multiplicity = list("single" = 0.97, "twins" = 0.03),
              birth_sex_ratio = list(male = 0.51, female = 0.49),
              death = list(yes = 0.1, no = 0.9),
              marriage_cohab_male = list(yes = 0.1, no = 0.9),
              marriage_no_cohab_male = list(yes = 0.1, no = 0.9),
              marriage_no_cohab_female = list(yes = 0.1, no = 0.9),
              separate_male = list(yes = 0.1, no = 0.9),
              separate_child_custody = list(male = 0.2, female = 0.8),
              separate_hhtype = list(lone = 0.5, group = 0.5),
              separate_hf_random_join = list("1" = 0.4, "2" = 0.3, "3" = 0.2, "4" = 0.1),
              divorce_male = list(yes = 0.5, no = 0.9),
              divorce_female = list(yes = 0.5, no = 0.9),
              cohabitation_male = list(yes = 0.1, no = 0.9),
              cohabitation_female = list(yes = 0.1, no = 0.9),
              breakup = list(yes = 0.1, no = 0.9),
              breakup_child_custody = list(male = 0.2, female = 0.8),
              breakup_hhtype = list(lone = 0.5, group = 0.5),
              breakup_hf_random_join = list("1" = 0.4, "2" = 0.3, "3" = 0.2, "4" = 0.1),
              leavehome_male = list(yes = 0.3, no = 0.7),
              leavehome_female = list(yes = 0.2, no = 0.8),
              leavehome_hhtype = list(lone = 0.2, group = 0.8),
              leavehome_hf_random_join = list("1" = 0.5, "2" = 0.3, "3" = 0.1, "4" = 0.1),
              migrant_individuals = dymiumCore::toy_individuals, 
              migrant_households = dymiumCore::toy_households)

for (i in seq_along(models)) {
  world$add(models[[i]], name = names(models)[i])
}

# matsim module's models and settings
matsim_runControler_model <-
  list(
    matsim_config = here::here("scenarios/matsim-integration/inputs/matsim/config.xml"),
    matsim_last_iteration = 1
  )
world$add(
  x = readRDS(here::here("scenarios/matsim-integration/inputs/models/vista_persons.rds")), 
  name = "vista_persons"
)
world$add(
  x = readRDS(here::here("scenarios/matsim-integration/inputs/models/vista_trips.rds")), 
  name = "vista_trips"
)
```

## Set up a simulation pipeline

We run the demographic events first then the MATSim events which are to
create travel demand (`event_matsim_createVISTADemand`) and to pass it
to MATSim (`event_matsim_runcontroler`). Note that, run controler wasn’t
added into `world` but instead we provide `matsim_runControler_model` in
the model argument of the function. MATSim will only be executed in the
time steps 1 and 5. `dm_save` saves the state of the world at that
moment to the scenario output directory, at
/var/folders/0d/9srpj\_750lxbkfs2\_8nwkcpw0000gn/T//RtmpyUVLNq/scenario/outputs
in this case.

``` r
for (i in 1:5) {
  world$start_iter(time_step = i, unit = "year") %>%
    event_demography_age$run(.) %>%
    event_demography_birth$run(.) %>%
    event_demography_death$run(.) %>%
    event_demography_cohabit$run(.) %>%
    event_demography_marriage$run(.) %>%
    event_demography_divorce$run(.) %>%
    event_demography_breakup$run(.) %>%
    event_demography_separation$run(.) %>%
    event_demography_migration$run(., target = 10) %>%
    event_matsim_createVISTADemand$run(.) %>%
    event_matsim_runcontroler$run(.,
                                  model = matsim_runControler_model,
                                  time_steps = c(1, 5),
                                  use_rJava = TRUE) %>%
    dm_save(.)
}
```

## Visualise the result

``` r
library(ggplot2)
library(patchwork)
library(scales)
history_data <- dymiumCore::get_history(world)
plot_history(x = world, by_entity = T)
```

``` r
log_data <- dymiumCore::get_log(world)
log_data %>%
  .[grepl("^cnt:|^avl:", desc), ] %>%
  .[, value := unlist(value)] %>%
  .[, group_label := gsub(".*:", "" , x = desc)] %>% 
  .[, group_label := gsub("_", " " , x = group_label)] %>% 
  .[, tag := gsub(":.*", "" , x = desc)] %>% .[] %>%
  ggplot(data = ., aes(x = time, y = value, group = desc, color = tag)) +
  geom_line() +
  facet_wrap(~ group_label, scales = "free", labeller = labeller(group_label = label_wrap_gen(25))) +
  scale_x_continuous(breaks = scales::pretty_breaks()) +
  scale_y_continuous(breaks = scales::pretty_breaks())
```

**cnt:** count of occurences

**avl:** count of the total number agents that were eligible to undergo
the event.
