create procedure syn.usp_ImportFileCustomerSeasonal
	@ID_Record int
as
set nocount on
begin
	declare @RowCount int = (select count(*) from syn.SA_CustomerSeasonal)                    #1.Для объявления переменных declare используется один раз.#
	declare @ErrorMessage varchar(max)                                                         #2.Дополнительное объявление переменных через declare используется только, если необходимо использовать ранее объявленную переменную для определения значения объявляемой. В данном случае в этом нет необходимости.#
                                                                                             #3.При объявлении более одной переменной за раз, они перечисляются через запятую каждая с новой строки с одним отступом.#
-- Проверка на корректность загрузки
	if not exists (
	select 1                                                                                 #4. В условных операторах весь блок кода смещается на 1 отступ.#
	from syn.ImportFile as f
	where f.ID = @ID_Record
		and f.FlagLoaded = cast(1 as bit)
	)
		begin                                                                                 #5. begin должен быть на том же уровне, что и  if not exists. #
			set @ErrorMessage = 'Ошибка при загрузке файла, проверьте корректность данных'   #6. Отступов перед set нет.#

			raiserror(@ErrorMessage, 3, 1)
			return                  #7.Перед return должна быть пустая строка.#
		end

	CREATE TABLE #ProcessedRows(ActionType varchar(255), ID int)         #8. Пробел нужен между скобкой и созданием объекта. # #9. Обязательность атрибута указывается сразу после типа данных.# #10. При создании объекта, запятые ставятся в конце строк и каждая переменная с новой строки# #11.Пустая строка после слов Create table.#
	
	--Чтение из слоя временных данных
	select
		cc.ID as ID_dbo_Customer
		,cst.ID as ID_CustomerSystemType
		,s.ID as ID_Season
		,cast(sa.DateBegin as date) as DateBegin
		,cast(sa.DateEnd as date) as DateEnd
		,cd.ID as ID_dbo_CustomerDistributor
		,cast(isnull(sa.FlagActive, 0) as bit) as FlagActive
	into #CustomerSeasonal
	from syn.SA_CustomerSeasonal cs                                     #12. Пропущено ключевое слово as.#
		join dbo.Customer as cc on cc.UID_DS = sa.UID_DS_Customer       #13. Все виды join указываются явно#
			and cc.ID_mapping_DataSource = 1                            #14.Дополнительные условия переносятся на следующую строку с 1 отступом#.
		join dbo.Season as s on s.Name = sa.Season
		join dbo.Customer as cd on cd.UID_DS = sa.UID_DS_CustomerDistributor
			and cd.ID_mapping_DataSource = 1
		join syn.CustomerSystemType as cst on sa.CustomerSystemType = cst.Name #15.Сперва указываем поле присоединяемой таблицы.#
	where try_cast(sa.DateBegin as date) is not null
		and try_cast(sa.DateEnd as date) is not null
		and try_cast(isnull(sa.FlagActive, 0) as bit) is not null

	-- Определяем некорректные записи
	-- Добавляем причину, по которой запись считается некорректной
	select
		sa.*
		,case                                                                                 #16. Запятую перед ключевым словом CASE перенести в предыдущую строку.
			when cc.ID is null then 'UID клиента отсутствует в справочнике "Клиент"'         #17.При написании конструкции с case, необходимо, чтобы when был под case с 1 отступом.#
			when cd.ID is null then 'UID дистрибьютора отсутствует в справочнике "Клиент"'
			when s.ID is null then 'Сезон отсутствует в справочнике "Сезон"'
			when cst.ID is null then 'Тип клиента в справочнике "Тип клиента"'
			when try_cast(sa.DateBegin as date) is null then 'Невозможно определить Дату начала'
			when try_cast(sa.DateEnd as date) is null then 'Невозможно определить Дату начала'
			when try_cast(isnull(sa.FlagActive, 0) as bit) is null then 'Невозможно определить Активность'
		end as Reason
	into #BadInsertedRows
	from syn.SA_CustomerSeasonal as cs
	left join dbo.Customer as cc on cc.UID_DS = sa.UID_DS_Customer                            #18. Все виды join пишутся с 1 отступом .#    
		and cc.ID_mapping_DataSource = 1
	left join dbo.Customer as cd on cd.UID_DS = sa.UID_DS_CustomerDistributor and cd.ID_mapping_DataSource = 1
	left join dbo.Season as s on s.Name = sa.Season
	left join syn.CustomerSystemType as cst on cst.Name = sa.CustomerSystemType
	where cc.ID is null
		or cd.ID is null
		or s.ID is null
		or cst.ID is null
		or try_cast(sa.DateBegin as date) is null
		or try_cast(sa.DateEnd as date) is null
		or try_cast(isnull(sa.FlagActive, 0) as bit) is null

		
end
