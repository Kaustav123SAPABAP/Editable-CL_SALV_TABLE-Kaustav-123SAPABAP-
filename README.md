# Editable-CL_SALV_TABLE
How to Create Editable ALV &amp; Perform Data Change by Pressing Enter
*&---------------------------------------------------------------------*
*& Report ZRODTEP_UTILIZATION
*&---------------------------------------------------------------------*
*&Author - Kaustav Ghosh
*&user wants a Report where he can edit the Fields and modify the report after putting some Data by Clicking enter.
*&---------------------------------------------------------------------*
report zrodtep_utilization.

types: begin of st_final,
         check(1)              type c,
         rodtep_scrip_no       type num10,
         rodtep_scrip_date     type char10,
         rodtep_scrip_exp_date type char10,
         rodtep_scrip_value    type char14,
         port_code             type char10,
         boe_no                type num7,
         boe_date              type char10,
         amount_debited        type char14,
         item                  type char50,
         asseseble_value       type char14,
         bcd_rate              type int3,
         bcd_value             type char14,
         balance_value         type char14,
       end of st_final.

data: it_final  type table of st_final,
      it_final1 type table of st_final,
      wa_final  type  st_final.

data lo_functions type ref to cl_salv_functions_list.
data lo_display type ref to cl_salv_display_settings.
data: lo_columns type ref to cl_salv_columns_table.
data: lo_column type ref to cl_salv_column_table.

data ls_key type salv_s_layout_key .
data: lo_header  type ref to cl_salv_form_layout_grid,
      lo_h_label type ref to cl_salv_form_label,
      lo_h_flow  type ref to cl_salv_form_layout_flow,
      lo_h_logo  type ref to cl_salv_form_layout_logo,
      lo_logo    type ref to cl_salv_form_layout_logo,
      lo_event   type ref to cl_salv_events_table.
data: lo_table type ref to cl_salv_table.
data: ls_layout         type lvc_s_layo,
      ls_fieldcat       type lvc_t_fcat,
      ls_modified_cells type lvc_s_moce.
data:lo_grid      type ref to cl_gui_alv_grid,
     lo_full_adap type ref to cl_salv_fullscreen_adapter.
field-symbols <fs_lo_fieldcat> like line of ls_fieldcat.
" Here we can print changed data for example
data: lt_mod_cells type lvc_t_modi,
      ls_mod_cells type lvc_s_modi,
      lv_message   type string.
field-symbols: <fs_mod_cells> type lvc_s_modi,
               <ft_mod_rows>  type table,
               <fs_mod_rows>  type any,
               <fs>           type any,
               <date>         type any,
               <scrip_value>  type any,
               <amount>       type any.




*  Define the Local class inheriting from the CL_SALV_MODEL_LIST
*  to get an access of the model, controller and adapter which inturn
*  provides the Grid Object
*----------------------------------------------------------------------*
class lcl_salv_model definition inheriting from cl_salv_model_list.
  public section.
    data: lo_control type ref to cl_salv_controller_model,
          lo_adapter type ref to cl_salv_adapter.
    methods:
      grabe_model
        importing
          io_model type ref to cl_salv_table,
      grabe_controller,
      grabe_adapter.
  private section.
    data: lo_model type ref to cl_salv_model.
endclass.

class lcl_ship definition.
  public section.
    data: lo_salv_model type ref to lcl_salv_model.
    methods get_data.
    methods build_columns changing lo_columns type ref to cl_salv_columns_table.
    methods set_text importing name       type lvc_fname
                               l_text     type scrtext_l
                     changing  lo_columns type ref to cl_salv_columns_table.
endclass.


start-of-selection.
  data(report) = new lcl_ship( ).
  report->get_data( ).



*----------------------------------------------------------------------*
* LCL_SALV_MODEL implementation
*----------------------------------------------------------------------*
class lcl_salv_model implementation.
  method grabe_model.
*   save the model
*   Get Model Object - narrow cast from cl_salv_table to cl_salv_model
    lo_model ?= io_model.
  endmethod.                    "grabe_model
  method grabe_controller.
*   save the controller
    lo_control = lo_model->r_controller.
  endmethod.                    "grabe_controller
  method grabe_adapter.
*   save the adapter from controller
    lo_adapter ?= lo_model->r_controller->r_adapter.
  endmethod.                    "grabe_adapter
endclass.

*local class definition for Handler
class lcl_handler definition.
  public section.

    methods: on_click for event link_click of cl_salv_events_table importing row column.
    methods: on_user_command for event added_function of cl_salv_events importing e_salv_function.
    methods:handle_data_changed for event data_changed of cl_gui_alv_grid importing er_data_changed.

endclass.

class lcl_handler implementation.

*Click on Field
  method on_click.
    read table it_final assigning field-symbol(<fin>) index row.
    check sy-subrc is initial.
    if <fin>-check is initial.
      <fin>-check ='X'.
    else.
      clear <fin>-check.
    endif..
    unassign <fin>.
    lo_table->refresh(   ).
  endmethod.

  method on_user_command.

    case e_salv_function.


*To select all field
      when 'SELECT'.
        loop at it_final assigning field-symbol(<fin>).
          <fin>-check = 'X'.
        endloop.
        message 'All data Selected' type 'S'.               "#EC NOTEXT
        lo_table->refresh(  ).

**to Deselect all field
      when 'DESELECT1'.
        unassign <fin>.
        loop at it_final assigning <fin>.
          <fin>-check = ''.
        endloop.
        message 'All data Unselected' type 'S'.             "#EC NOTEXT
        lo_table->refresh(  ).


      when 'EDIT'.

* contorller
        call method report->lo_salv_model->grabe_controller.
* Adapter
        call method report->lo_salv_model->grabe_adapter.
* Fullscreen Adapter (Down Casting)
        lo_full_adap ?= report->lo_salv_model->lo_adapter.
* Get the Grid
        lo_grid = lo_full_adap->get_grid( ).
* Register edit event
        lo_grid->register_edit_event( exporting i_event_id = cl_gui_alv_grid=>mc_evt_enter ).
        lo_grid->register_edit_event( exporting i_event_id = cl_gui_alv_grid=>mc_evt_modified ).
        set handler handle_data_changed for lo_grid.



*       Got the Grid .. ?
        if lo_grid is bound.

          call method lo_grid->get_frontend_fieldcatalog
            importing
              et_fieldcatalog = ls_fieldcat.

          loop at ls_fieldcat assigning <fs_lo_fieldcat>.
            if <fs_lo_fieldcat>-fieldname = 'RODTEP_SCRIP_NO'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'RODTEP_SCRIP_DATE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'RODTEP_SCRIP_VALUE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'PORT_CODE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'BOE_NO'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'BOE_DATE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'AMOUNT_DEBITED'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'ITEM'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'ASSESEBLE_VALUE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'BCD_RATE'.
              <fs_lo_fieldcat>-edit = 'X'.
            elseif   <fs_lo_fieldcat>-fieldname = 'BCD_VALUE'.
              <fs_lo_fieldcat>-edit = 'X'.
            endif.

          endloop.


          call method lo_grid->set_frontend_fieldcatalog
            exporting
              it_fieldcatalog = ls_fieldcat.

*         refresh the table
          call method lo_grid->refresh_table_display.
        endif.


    endcase.
  endmethod.

  method handle_data_changed.



    assign er_data_changed->mp_mod_rows->* to <ft_mod_rows>.
    " Since we call data change event immediately after changing cell
    " objects mp_mod_rows and mt_mod_cells always have 1 record
    " so we read just at index 1

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'RODTEP_SCRIP_NO' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'RODTEP_SCRIP_DATE' of structure <fs_mod_rows> to <fs>.
    assign component 'RODTEP_SCRIP_DATE' of structure <fs_mod_rows> to <date>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'RODTEP_SCRIP_VALUE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'PORT_CODE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'BOE_NO' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'BOE_DATE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'AMOUNT_DEBITED' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'ITEM' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'ASSESEBLE_VALUE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'BCD_RATE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.

    read table <ft_mod_rows> assigning <fs_mod_rows> index 1.
    assign component 'BCD_VALUE' of structure <fs_mod_rows> to <fs>.
    lt_mod_cells = er_data_changed->mt_mod_cells.


    read table lt_mod_cells into data(l_mod_cells) index 1.
    data(ind) = l_mod_cells-row_id.


    data: prev_date like sy-datum.
    data: scrip_date like sy-datum.

    scrip_date = |{ <date>+06(04) }{ <date>+03(02) }{ <date>+00(02) }|.

    if scrip_date is not initial.
      call function 'RP_CALC_DATE_IN_INTERVAL'
        exporting
          date      = scrip_date
          days      = 0
          months    = 0
          signum    = '-'
          years     = 2
        importing
          calc_date = prev_date.
    endif.

    read table it_final assigning field-symbol(<fin>) index ind.
    if <fin> is assigned.
      move-corresponding <fs_mod_rows> to <fin>.
      <fin>-rodtep_scrip_exp_date =  |{ prev_date+06(2) }.{ prev_date+04(02) }.{ prev_date+00(04) }|.
      <fin>-balance_value = <fin>-rodtep_scrip_value - <fin>-amount_debited.
      modify it_final from <fin> index ind.
    endif.

    lo_table->refresh( ).


    call method lo_grid->refresh_table_display.


    lv_message = |Data Changed|.
    message lv_message type 'S'.

  endmethod.

endclass.
class lcl_ship implementation.

  method get_data.

    do 200 times.
      append wa_final to it_final.
    enddo.

    try.
        call method cl_salv_table=>factory
          importing
            r_salv_table = lo_table
          changing
            t_table      = it_final.
      catch cx_salv_msg into data(lo_err).
        message id lo_err->msgid type lo_err->msgty number lo_err->msgno with lo_err->msgv1 lo_err->msgv1 lo_err->msgv1 lo_err->msgv1.
        return.
    endtry.

    ls_key-report = sy-repid.
    lo_table->get_layout( )->set_key( value = ls_key ).

    lo_display = lo_table->get_display_settings( ).
    lo_display->set_striped_pattern( cl_salv_display_settings=>true ). "zebra
    lo_display->set_list_header( 'SALV Full Template' ). "gui_title
    lo_display->set_horizontal_lines( cl_salv_display_settings=>false ).
    lo_display->set_vertical_lines( cl_salv_display_settings=>true ).
    lo_display->set_list_header( |Number of displayed rows: { lines( it_final ) }| ).
    "*-----------------Selection-------------------------*
    data: lo_selections type ref to cl_salv_selections.
    lo_selections = lo_table->get_selections( ).
    lo_selections->set_selection_mode( if_salv_c_selection_mode=>row_column ).
    lo_table->get_layout( )->set_save_restriction( if_salv_c_layout=>restrict_none ).
    lo_functions = lo_table->get_functions( ).
    lo_functions->set_default( abap_true ). 
    lo_functions->set_all( abap_true ).
    "*-----------------Columns - Fieldcatalog-------------------------*
    lo_columns = lo_table->get_columns( ).
    lo_columns->set_optimize( abap_true ).
    lo_columns->set_key_fixation( abap_true ).
    build_columns( changing lo_columns = lo_columns ).

    try.
        lo_column ?= lo_columns->get_column( 'CHECK' ).     "#EC NOTEXT
        lo_column->set_short_text( 'Checkbox' ).            "#EC NOTEXT
        lo_column->set_medium_text( 'Checkbox' ).           "#EC NOTEXT
        lo_column->set_long_text( 'Checkbox' ).             "#EC NOTEXT
        lo_column->set_cell_type( if_salv_c_cell_type=>checkbox_hotspot ).
        lo_column->set_output_length( 10 ).

      catch cx_salv_not_found into data(not_found).
    endtry.

*header object
    create object lo_header.

*information in Bold
    lo_h_label = lo_header->create_label( row = 1 column = 1 ).
    lo_h_label->set_text( `Rodtep Utilization Report` ).    "#EC NOTEXT

    describe table it_final lines data(line).
*information in tabular format
    lo_h_flow = lo_header->create_flow( row = 3  column = 1 ).
    lo_h_flow->create_text( text = `Date - ` ).             "#EC NOTEXT

    lo_h_flow = lo_header->create_flow( row = 3  column = 2 ).
    lo_h_flow->create_text( text = sy-datum ).

    lo_h_flow = lo_header->create_flow( row = 3  column = 5 ).
    lo_h_flow->create_text( text = `User - `  ).            "#EC NOTEXT

    lo_h_flow = lo_header->create_flow( row = 3  column = 7 ).
    lo_h_flow->create_text( text = sy-uname ).


    lo_h_flow = lo_header->create_flow( row = 4  column = 1 ).
    lo_h_flow->create_text( text = `Time - ` ).             "#EC NOTEXT

    lo_h_flow = lo_header->create_flow( row = 4  column = 2 ).
    lo_h_flow->create_text( text = sy-uzeit ).

    lo_h_flow = lo_header->create_flow( row = 4  column = 5 ).
    lo_h_flow->create_text( text = `Line - ` ).             "#EC NOTEXT

    lo_h_flow = lo_header->create_flow( row = 4  column = 7 ).
    lo_h_flow->create_text( text = line ).


*logo object
    create object lo_logo.

* Set left content
    lo_logo->set_left_content( lo_header ).

* set Right Image
    lo_logo->set_right_logo( `LOGO` ).                      "#EC NOTEXT

*set the top of list using the header for Online.
    lo_table->set_top_of_list( lo_logo ).
    lo_table->set_top_of_list_print( lo_header ).

*set PF status
    if it_final is not initial.
      set pf-status 'ZPFSTATUS'.
    endif.

*Get PF Status
    lo_table->set_screen_status(
      pfstatus = 'ZPFSTATUS'
      report   = sy-repid
      set_functions = lo_table->c_functions_all ).

*Event Handle For PF status
    data: lo_handler type ref to lcl_handler.
    create object lo_handler.
    lo_event = lo_table->get_event(  ).

*Event for Checkbox
    set handler lo_handler->on_click  for lo_event.
*Event for User Command
    set handler lo_handler->on_user_command for lo_event.


*   object for the local inherited class from the CL_SALV_MODEL_LIST
    create object lo_salv_model.
*   grabe model to use it later
    call method lo_salv_model->grabe_model
      exporting
        io_model = lo_table.


    lo_table->display( ).

  endmethod.


  method build_columns.

    "*-----------------Set Column Texts-------------------------*

    set_text( exporting name = 'RODTEP_SCRIP_NO'       l_text = 'Rodtep Scrip No.'            changing lo_columns = lo_columns ).
    set_text( exporting name = 'RODTEP_SCRIP_DATE'     l_text = 'Rodtep Scrip Date'           changing lo_columns = lo_columns ).
    set_text( exporting name = 'RODTEP_SCRIP_EXP_DATE' l_text = 'Rodtep Scrip Expiry Date'    changing lo_columns = lo_columns ).
    set_text( exporting name = 'RODTEP_SCRIP_VALUE'    l_text = 'Rodtep Scrip Value)'         changing lo_columns = lo_columns ).
    set_text( exporting name = 'PORT_CODE'             l_text = 'Port Code'                   changing lo_columns = lo_columns ).
    set_text( exporting name = 'BOE_NO'                l_text = 'Bill of Entry No.'           changing lo_columns = lo_columns ).
    set_text( exporting name = 'BOE_DATE'              l_text = 'Bill of Entry Date'          changing lo_columns = lo_columns ).
    set_text( exporting name = 'AMOUNT_DEBITED'        l_text = 'Amount Debited'              changing lo_columns = lo_columns ).
    set_text( exporting name = 'ITEM'                  l_text = 'Item Description'            changing lo_columns = lo_columns ).
    set_text( exporting name = 'ASSESEBLE_VALUE'       l_text = 'Assesseble Value'            changing lo_columns = lo_columns ).
    set_text( exporting name = 'BCD_RATE'              l_text = 'BCD Rate'                    changing lo_columns = lo_columns ).
    set_text( exporting name = 'BCD_VALUE'             l_text = 'BCD Value'                   changing lo_columns = lo_columns ).
    set_text( exporting name = 'BALANCE_VALUE'         l_text = 'Balance Value of RoDTEP '    changing lo_columns = lo_columns ).


  endmethod.


  method set_text.


    try.
        lo_column ?= lo_columns->get_column( name ).
        if l_text is not initial.
          lo_column->set_long_text( l_text ).
          lo_column->set_fixed_header_text( `L` ).
        endif.

      catch cx_salv_not_found into data(err).
    endtry.

  endmethod.


endclass.

